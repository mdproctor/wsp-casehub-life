---
layout: post
title: "Why your LLM workers shouldn't be making phone calls"
date: 2026-10-27
entry_type: article
subtype: diary
projects: [casehubio/life]
tags: [playbooks, serverless-workflow, architecture, connector-coordination, design-decision, llm-efficiency]
series: issue-140-playbook-coordination
---

# Why your LLM workers shouldn't be making phone calls

A home renovation case touches contractors, calendars, documents,
locations, and finance. In casehub-life, each of those domains has an
LLM-backed worker — an agent with a system prompt, a response schema,
and access to connector platform tools via MCP. The worker calls
`search_contacts` to find the contractor, `search_places` to geocode
the site, `list_transactions` to pull payment history. Each call is a
round-trip through the LLM: the model reasons about what to look up,
formulates the tool call, waits for the result, processes it, and
moves on.

Every one of those lookups is mechanical. The LLM isn't making a
judgment call when it searches for "Acme Plumbing" in the contacts
database. It's doing clerical work at the price of reasoning.

![Before and after — LLM round-trips vs playbook pre-staging](images/worker-before-after.svg)

## The connector surface

After wiring four connector SPIs last week — ContactsPlatform,
EmailPlatform, DocumentPlatform, LocationPlatform — casehub-life
has access to over 100 GraphQL operations across 12 connector
domains. HealthPlatform, TravelPlatform, and CommercePlatform landed
the same week. Every operation is exposed two ways: as an MCP tool
(for LLM workers to call mid-task) and as a REST endpoint via
`@McpDomain` annotations (for programmatic access).


![The connector surface — 12 domains, 100+ operations, two access paths](images/connector-surface.svg)

The dual-path architecture means the same operations are available
for both paths. An LLM worker calling `search_contacts` mid-task is
making a single lookup. A playbook calling `/api/connectors/contacts/searchContacts`
over REST is doing the same thing — without an LLM in the loop.

## The execution model

The idea is straightforward. An LLM should plan the coordination, not
execute it. The judgment call is "what does this situation need?" — which
playbooks to run, which connectors to hit, what post-processing to do
after workers finish. The mechanical work — actually calling the APIs,
batching the results, combining them into a context package — that's
deterministic. No tokens required.

![Playbook execution pipeline — authoring format to SWF execution engine](images/playbook-execution-pipeline.svg)

Four phases, two of which are LLM-free:

**Phase 1 — Planning.** A coordination worker (AgentExec) assesses the
incoming request, selects pre-processing and post-processing playbooks
from a catalog, and writes a `CoordinationPlan` to the case context.
This is the one LLM call that earns its tokens — it's making decisions
about what this specific case needs.

**Phase 2 — Pre-processing.** A playbook runs: search contacts, geocode
the site, pull recent transactions, check calendar availability. All
`rest-request:` steps, executed mechanically. The domain worker receives
a pre-assembled context package and can focus on judgment — "is this
quote reasonable given the contractor's trust score?" instead of "let me
look up the contractor first."

**Phase 3 — Domain workers.** The existing AgentExec workers, unchanged.
They still have MCP tool access for in-flight decisions the playbook
couldn't anticipate. But the routine lookups are already done.

**Phase 4 — Post-processing.** Another playbook: store the completion
report in DocumentPlatform, email a summary, send a chat notification,
update the calendar. Sequenced, mechanical, no LLM needed.

## Why playbooks, not raw Serverless Workflow

This is where the design session got interesting.

I started with what seemed obvious: define the mechanical work as
Serverless Workflow `do:` blocks and execute them through the existing
`FlowWorkerFunction`. The platform already had the CNCF SWF SDK-Java
wired up. Build a dispatcher to route `call:` steps to connector
APIs, add some plumbing to the engine — done.

Three problems surfaced during the design review. `PlanItemDefinition`
was sealed. `CallableDispatchRegistry` did exact-match lookup only.
`CasehubFlowContext.callableRegistry()` was package-private. Every
assumption required breaking an API boundary in a cross-repo
dependency. The review pushed toward raw SWF as the solution, which
would have meant writing 38-line YAML blocks for what should be 11
lines of coordination logic.

The answer had been in the platform all along. CaseHub has a playbook
language — built during the SC2 build-order work, extended through
platform #562 and #563. `PlaybookCompiler` compiles playbook YAML into
the same SWF `Workflow` objects that the SDK-Java executes. Same
engine, better language.

![Playbook vs SWF 1.0 comparison — 11 lines vs 38 lines for the same behaviour](images/playbook-vs-swf-comparison.svg)

The playbook language is strictly more expressive than raw SWF YAML.
`retry: 3` as an inline decorator vs nested `TryTask → RetryPolicy →
RetryLimit → RetryLimitAttempt`. `parallel:` for concurrent operations
vs `fork → branches → do` nesting. `signal:` and `await:` for
coordination primitives that have no concise SWF equivalent. The
compilation is additive — everything SWF can express, a playbook can
express. The reverse isn't true.

And the architectural payoff: no engine changes at all. Playbooks use
`rest-request:` steps for connector calls — direct HTTP to the REST
endpoints. `PlaybookCompiler` compiles them, `FlowWorkerFunction` wraps
the result, `FlowWorkerFunctionHandler` executes it. Zero new platform
SPIs, zero cross-repo modifications.

## Playbook-first, universally

SWF 1.0 is a good execution engine. CaseHub uses it — under the hood,
invisible. But as an authoring format for domain coordination, it's the
wrong level of abstraction. You don't want to write `TryTask` with
`RetryPolicy` when you mean `retry: 3`.

The playbook language exists because build-order coordination, event
sequencing, and multi-step orchestration all need the same primitives:
parallel execution, conditional branching, retry with backoff, timeout,
signals, barriers. The SC2 work needed them. IoT monitoring needs them.
AML investigation workflows need them. And now connector coordination
in casehub-life needs them.

If interop matters, `PlaybookCompiler` can emit SWF YAML as an export
format. The types are the same. But the authoring direction is settled:
playbooks everywhere, SWF underneath.

## What's being built

The implementation plan has two batches:

**Foundation:** `CoordinationPlan` record, `CoordinationTemplate` JPA
entity with runtime store, `CoordinationTemplateCatalog` that merges
curated classpath playbooks with LLM-generated ones,
`CoordinationTemplateValidator` for security (URL allowlisting on
generated playbooks), and 8 curated `.playbook.yaml` templates covering
contractor onboarding, appointment prep, financial review, travel
research, and three post-processing patterns.

**Case integration:** `buildFreshDefinition()` on `LifeTypedCaseHub`
that constructs per-case `CaseDefinition` instances with
`FlowWorkerFunction` workers compiled from the selected playbooks. An
extended 5-phase `LifeCaseService.startCase()` — the existing scaffold →
CBR retrieval flow gains a planning worker invocation and definition
build before engine start.

No cross-repo changes. The platform already has everything this needs.
