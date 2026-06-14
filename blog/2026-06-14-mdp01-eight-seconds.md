---
layout: post
title: "Eight Seconds"
date: 2026-06-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [openclaw, testcontainers, podman, layer-7]
---

Layer 7's workers don't exist yet. Before I write them, I wanted to know the tests could be written — that there's a pattern that actually works for spinning up OpenClaw in a container, handing it a mock LLM, and verifying the call chain without hitting real APIs.

So we built a mini project to prove it. What followed was three days of progressively more exotic problems.

The first wall was Podman. macOS Podman uses an applehv VM, and the socket Testcontainers needs lives inside that VM. No `/var/run/docker.sock`, and the machine-specific API socket is a symlink that goes stale. We got past it with an SSH tunnel — forward the VM's internal Podman socket to a stable local path, point Testcontainers at it. Ugly, but it works.

The second wall arrived when we finally started a container and it immediately OOMKilled. The Podman machine has 4GB. That should be fine — Node.js processes need headroom, but 4GB is plenty. Except there were 91 postgres containers still running from old Quarkus dev service sessions, quietly eating most of it. Cleaned those out, and OpenClaw started.

The third wall was the networking. I'd set up an `--internal` Docker network for isolation — containers can reach each other, but no internet egress. OpenClaw OOMKilled on that too, immediately, without explanation. Podman's applehv VM handles `--internal` networks through a different cgroup path, and something about that interacts badly with Node.js memory accounting. Dropped the `--internal` flag and moved on. Internet isolation on macOS is enforced at the provider level instead — the only LLM provider configured is WireMock, so real API calls simply have nowhere to go.

That left the port forwarding problem. Testcontainers assigns a random host port, `getMappedPort()` tells you what it is, you curl it — except containers on Testcontainers-created custom networks in Podman return `Invalid Upgrade header` on every HTTP request. The port is bound, the container is running, but the applehv VM's networking layer is responding instead of the container. The fix was to stop making HTTP calls from the host entirely. Everything goes through `execInContainer` — container-to-container via network aliases, no host port mapping involved.

Once that was sorted, the wrong docs were waiting. `POST /api/sessions/main/messages` — widely cited, confirmed by multiple integration guides — returns 404. So does every `/api/*` path, `/hooks/agent`, `/hooks/wake`. The actual way to trigger an agent turn is `POST /v1/chat/completions`, but you have to enable it in the config (`gateway.http.endpoints.chatCompletions.enabled: true`), and the `model` field must be `"openclaw"` — not the upstream provider model. OpenClaw is the model from the caller's perspective; it routes to the real provider internally.

```java
.waitingFor(Wait.forSuccessfulCommand("curl -sf http://localhost:18789/healthz")
    .withStartupTimeout(Duration.ofMinutes(3)))
```

```json
{"model": "openclaw", "messages": [{"role": "user", "content": "ping"}]}
```

```
ChatCompletion: exit=0 out={"model":"openclaw","choices":[{"message":{"content":"pong"}}]}
WireMock intercepted 1 LLM calls
Tests run: 3, Failures: 0, Errors: 0 — Total time: 8.587s
```

That `"pong"` is the response configured in WireMock. OpenClaw received the message, called our mock LLM, got `"pong"` back, and returned it. The whole chain — casehub-life test → OpenClaw container → WireMock → back through OpenClaw → response — in 8 seconds.

The mini project is at `/tmp/openclaw-isolation-test/` and passes. The critical path is confirmed: engine#463 (function-as-worker design) → casehub-openclaw (WorkerProvisioner implementation) → life#25 (wire up real workers). The test infrastructure is ready to receive them.
