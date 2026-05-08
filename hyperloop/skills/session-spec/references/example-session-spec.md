> **[Guidance — Metadata table]** If `/session-spec` invoked with GitHub issue URL, metadata table written **immediately after H1** and **before `## Goal`**, as shown below. No issue URL → omit table; H1 followed directly by `## Goal`.
>
> **Table-present example** (issue URL detected):
>
> ```markdown
> # Notification Delivery
>
> | Field        | Value                           |
> | ------------ | ------------------------------- |
> | Source Issue | samrom3/claude-hyper-plugs#42   |
>
> ## Goal
> ```
>
> **Table-absent example** (no issue URL — default):
>
> ```markdown
> # Notification Delivery
>
> ## Goal
> ```

# Notification Delivery

| Field        | Value                         |
| ------------ | ----------------------------- |
| Source Issue | samrom3/claude-hyper-plugs#42 |

## Goal

Deliver notifications to users when important events occur — routing messages through configurable channels (email, in-app, webhook) without producers knowing delivery details.

## Context

- No existing notification infrastructure. Background processes complete silently today.
- `CLAUDE.md` ADR scan found no conflicting decisions on channel routing.
- `NotificationPayload` must be immutable (value object pattern per existing domain model).

## Non-goals

- No retry/queue infrastructure (callers retry at own layer).
- No admin UI for managing notification templates.
- No rate limiting or deduplication.

## Steps

> **Scaffold-first note:** For new API surface, first step creates typed stubs with failing tests. Subsequent steps implement business logic against those stable contracts via TDD.

### STEP-notification-delivery-01: Scaffold NotificationPayload and DeliveryService stubs

Create `NotificationPayload` value object (`src/models/`) and `DeliveryService` stub (`src/services/`) with `NotImplementedError` body. Add `PayloadBuilder` with chained construction. Write skeleton tests covering expected API shape (tests will fail on `NotImplementedError`).

→ verify: `pytest tests/test_notification_delivery.py` exits non-zero (stubs exist, tests fail as expected on NotImplementedError); `from src.models.notification import NotificationPayload` imports without error

### STEP-notification-delivery-02: Implement DeliveryService routing logic

Implement `DeliveryService.deliver()` to route by `NotificationPayload.channel`. Handle unknown channel with explicit error; reject empty body. All scaffold tests must now pass.

→ verify: `pytest tests/test_notification_delivery.py` exits 0; `grep -c "NotImplementedError" src/services/delivery.py` equals 0

## Open Questions

- OQ-1: Should unknown-channel errors log a warning or raise immediately?
- OQ-2: Is there an existing `User` model, or should `recipient_id` be a plain string for V1?
