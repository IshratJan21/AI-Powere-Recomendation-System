# ADR-002: FastAPI as Backend Framework

**Status:** Proposed
**Date:** 2026-07-24

## Context
The system is AI-heavy: Google Gemini and OpenAI SDKs are Python-native. Structured-output validation aligns with Pydantic.

## Decision
Adopt **FastAPI + Pydantic v2 + Uvicorn** as the backend platform, with **Arq** for async workers.

## Consequences
+ Native async, high throughput per worker.
+ Pydantic doubles as our schema layer (edge validation + AI output validation).
+ First-class OpenAPI generation drives the typed TS client.
- Python GIL — CPU-bound work runs in workers, not request handlers.
