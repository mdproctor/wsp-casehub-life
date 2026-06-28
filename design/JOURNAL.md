# Design Journal — issue-37-layer7-worker-provisioner

## §1 Architecture Decision: Two-Mode Coexistence

life#37 was originally scoped to replace all inline workers with provisioner-based workers. After deep analysis of the engine's execution paths (verified via bytecode), we determined the two modes serve fundamentally different needs:

- **AgentExec (inline)**: synchronous request/response, structured WorkerResult, direct case context update
- **Provisioner (heartbeat)**: periodic monitoring, Quartz-scheduled, CaseHubRuntime.signal() delivery

The engine's natural fallthrough (inline match → provisioner) enables coexistence without configuration. All 32 AgentExec workers (life#38) remain unchanged. 7 new sentinel capabilities use the provisioner path.

## §2 Critical Findings That Shaped the Design

Three verified findings drove major design pivots:

1. **QhorusMessageSignalBridge drops STATUS** — rev 2 proposed channel-based delivery via STATUS messages. Bytecode verification showed the bridge's `isCommitmentResolving()` whitelist silently drops STATUS. Pivoted to direct `CaseHubRuntime.signal()`.

2. **OpenClawAgentRegistry 1:1 constraint** — `register()` uses `put()` not `putIfAbsent()`. Concurrent same-agent cases silently overwrite. Created `LifeSentinelRegistry` with (caseId, capability) keying instead.

3. **Engine never calls terminate()** — no call site exists in engine runtime. Created `LifeProvisionerCleanupObserver` to handle termination via `CaseLifecycleEvent`.

## §3 Sentinel Architecture

The sentinel pattern: provisioner registers in LifeSentinelRegistry (idempotent guard prevents re-firing), schedules a Quartz LifeHeartbeatJob. Each tick: CaseHubRuntime.query() for fresh case context → Agent.execute() via DirectCallBridge → CaseHubRuntime.signal("sentinelReport") → case plan bindings react to `.sentinelReport.escalationRequired`. LifeProvisionerCleanupObserver terminates sentinels on CaseCompleted/CaseFaulted/CaseCancelled.

Zero new CDI bean activations. Zero openclaw-casehub beans un-excluded. Sentinels reuse the proven AgentExec infrastructure (DirectCallBridge + LifeOpenClawChatModelFactory).
