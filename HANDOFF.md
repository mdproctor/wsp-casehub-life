*Updated: #77 closed — removed from What's Left.*

# Handoff — 2026-07-26

Closed #60 (OpenClaw skill integration) and #77 (engine + platform SNAPSHOT API adaptation). Single squashed commit 9b0210e on main.

## Last Session

Two-tier skill model (NATIVE/OPENCLAW) for external service integration. All 32 worker prompts and 7 sentinel prompts upgraded to tool-aware — reference calendar_create_event, iot_get_state, bank_get_transactions, send_chat. 39 response schemas gain tool-derived fields and toolsUsed. Design spec adversarially reviewed (4 rounds, 14 issues). Fixed pre-existing SNAPSHOT API breaks: SettingsScope.of→Path, WorkerExecutionContext removed (engine#693), WorkerFunction<T,R> two-param.

## What's Left

- parent#378 — docs/repos/casehub-life.md needs life-ui module section · S · Low

## Cross-Module

**Enabled** (we delivered, downstream work unblocked):
- `connectors` — CalendarPlatform SPI follows ChatPlatform pattern (connectors#88) · M · Med
- `iot` — MCP tool exposure for DeviceProvider (iot#69) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#88 | CalendarPlatform SPI + Google Calendar provider | M | Med | Follows chat-spi pattern |
| iot#69 | MCP tool exposure for DeviceProvider | S | Low | SPI already exists |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
