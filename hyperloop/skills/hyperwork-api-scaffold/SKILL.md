---
name: hyperwork-api-scaffold
description: API scaffolding — generate route/handler/schema/test skeletons for new endpoints, following existing project patterns.
user-invocable: false
---

# API Scaffold

**Use:** Tasks that create new API endpoints or services from scratch.

## Approach

1. Read existing endpoint files before writing anything — match patterns exactly.
2. Identify: router/handler pattern, request validation library, response shape, error format.
3. Scaffold in order: schema/types → handler → route registration → tests.
4. Tests: cover happy path + at least one 4xx (bad input) + one 4xx (not found/auth).

## Output Checklist

- Request schema with validation (not raw `req.body` — use zod/pydantic/joi per repo).
- Response type explicit — no `any` or `dict`.
- Error responses follow existing error shape (read existing handlers for format).
- Route registered in correct router/router group.
- OpenAPI/docstring annotation if repo uses them.

## Do / Don't

- DO: grep for an existing similar endpoint and copy its structure — consistency > novelty.
- DO: include auth middleware if all other routes have it.
- DON'T: invent new patterns — match what's there.
- DON'T: skip validation — unvalidated endpoints are a security boundary violation.
- DON'T: hardcode env vars — use config injection pattern from repo.
