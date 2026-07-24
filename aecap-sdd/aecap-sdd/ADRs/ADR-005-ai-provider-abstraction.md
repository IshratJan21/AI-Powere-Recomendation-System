# ADR-005: AI Provider Abstraction Layer

**Status:** Proposed
**Date:** 2026-07-24

## Context
Gemini is our primary model; OpenAI is a strong fallback. Providers, prices, and capabilities change every few months. We must avoid vendor lock-in and support per-tenant routing.

## Decision
Define an `AIProviderPort` in the application layer with adapters for Gemini and OpenAI, fronted by a `ProviderRouter` that applies per-tenant policy (primary/fallback, budget cap, latency target). Use circuit breakers (pybreaker) and typed retries (tenacity). All calls emit token/latency/cost metrics.

## Consequences
+ Model swaps are a config change, not a refactor.
+ Graceful degradation when a provider fails.
+ Enables A/B and canary of models per prompt version.
- Adds an indirection layer; contract tests are mandatory to keep adapters honest.
