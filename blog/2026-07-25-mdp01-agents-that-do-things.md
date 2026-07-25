---
layout: post
title: "Agents That Do Things"
date: 2026-07-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [openclaw, skill-integration, mcp, architecture]
series: issue-60-openclaw-skill-integration
---

*Part of a series on [#60 — OpenClaw skill integration](https://github.com/casehubio/life/issues/60).*

The eight case plans in casehub-life have had LLM-backed workers since Layer 7. Thirty-two of them across seven CaseHubs, plus seven sentinel heartbeats monitoring progress on active cases. Each gets a system prompt, reasons about the input, and produces structured output.

None of them could actually *do* anything. A home maintenance agent could assess that the property needs inspection and produce a JSON report saying so. It couldn't read a sensor, schedule a calendar event, or send a message to the contractor. The OpenClaw agents were thinkers, not actors.

This branch changes that. Every worker and sentinel prompt now references the MCP tools it should use — `iot_get_state` for sensor data, `calendar_create_event` for scheduling, `send_chat` for messaging, `bank_get_transactions` for financial data. The response schemas gain fields for tool-derived data: actual calendar event IDs, real sensor readings, notification message IDs. When the agents run via OpenClaw's `/hooks/agent`, they now have a tool palette.

The interesting design question wasn't *how* to wire tools — that's configuration. It was *where* the tools should live. Three options surfaced: build everything as CaseHub-native MCP tools, use OpenClaw's community skill ecosystem for everything, or hybrid.

The answer turned out to be layered. CaseHub already has the SPI bank for two of the four domains — connectors owns `ChatPlatform` (Slack, Discord, WhatsApp, SMS, Teams, Email), and the iot repo has `DeviceProvider` (Home Assistant, OpenHAB). Calendar needs a new `CalendarPlatform` SPI in connectors, following the same pattern as chat. Banking stays OpenClaw-only until usage patterns clarify what needs native treatment.

The key distinction is what CaseHub adds *around* the API call. A native MCP tool runs inside the Quarkus app — it has `CurrentPrincipal` for RBAC, `LedgerEntryRepository` for audit, `TrustGateService` for trust scoring, and `LifeGdprErasureService` for GDPR scope. An OpenClaw community skill has none of that. The platform properties are per-call for native tools; they're turn-level at best for OpenClaw tools.

That makes promotion criteria concrete: when a tool needs RBAC enforcement, per-call audit, trust scoring, or GDPR scope, it gets promoted from OpenClaw to native. Same tool name, CaseHub MCP endpoint takes priority. The agent never knows.

Along the way I fixed three pre-existing SNAPSHOT API breaks that had been blocking compilation — `SettingsScope.of()` gained a `Path` parameter, `WorkerExecutionContext` was removed entirely (engine#693 — zero ThreadLocal in the engine now), and `WorkerFunction` gained a second type parameter. The `CbrInputTransformer` needed reworking since it was reading experiences from the deleted ThreadLocal.

The spec went through four rounds of adversarial design review. The reviewer caught real issues — `toolsUsed` being LLM self-reported is architecturally wrong for trust scoring (downgraded to UI convenience), the schema examples used fictional type names, and the MCP config mechanism was underspecified. Fourteen issues raised, all resolved.

The two-tier model should generalise. Any CaseHub application that integrates with external services faces the same question: native for accountability properties, OpenClaw for the long tail. The SPI bank — connectors for communication, iot for devices, future modules for calendar and banking — is the platform's answer to that question.
