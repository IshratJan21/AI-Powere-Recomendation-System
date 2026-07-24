# ADR-007: JSON Schema as the Structured Output Contract

**Status:** Proposed
**Date:** 2026-07-24

## Context
Downstream automations (n8n workflows, CRM push, Sheets, webhooks) must never receive malformed AI output.

## Decision
Every AI extraction is governed by a **versioned JSON Schema** stored in `packages/schemas/`. LLM calls use provider-side structured output (`responseSchema` / `json_schema`). Server re-validates via Pydantic. Invalid outputs enter a **self-repair loop** (max 2 attempts) before quarantine to DLQ.

## Consequences
+ Contract is code — CI fails on drift.
+ Prompts and schemas are versioned together (`extract@2.4.1`).
+ Auditors can inspect the exact envelope any given action was built from.
- Schema evolution requires expand/migrate/contract discipline.
