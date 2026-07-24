# ADR-001: Modular Monolith over Microservices

**Status:** Proposed
**Date:** 2026-07-24

## Context
We need clear domain boundaries without the operational cost of running many services at the current team & tenant scale.

## Decision
Ship a **modular monolith** using Clean/Hexagonal Architecture. Each bounded context (ingestion, ai, actions, tenant, audit) lives in its own module with a public port interface. Any module can be extracted into its own service later without refactoring.

## Consequences
+ Single deploy, single log stream, cheap ops.
+ Strong internal boundaries enforced by import lint rules.
- Requires discipline: no cross-module imports outside `ports/`.
- Scaling is per-process until extraction.
