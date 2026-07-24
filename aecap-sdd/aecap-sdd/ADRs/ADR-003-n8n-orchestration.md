# ADR-003: n8n for Business Action Orchestration

**Status:** Proposed
**Date:** 2026-07-24

## Context
Business actions (CRM push, Sheets, Slack, Calendar, webhooks) are branching, retry-heavy, and change frequently. Ops/CS teams need visibility.

## Decision
Use **self-hosted n8n on Railway** as the orchestration runtime. FastAPI emits a signed job to n8n after extraction; n8n handles fan-out, retries, and human-in-the-loop steps.

## Consequences
+ Visual workflows are auditable and editable by non-engineers.
+ Built-in retry/back-off, credential storage, HTTP nodes.
- Adds one more moving piece; workflows must be exported as JSON and versioned in the repo.
