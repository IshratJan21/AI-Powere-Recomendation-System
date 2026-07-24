# ADR-006: Postgres-backed Event Bus (Outbox + LISTEN/NOTIFY)

**Status:** Proposed
**Date:** 2026-07-24

## Context
We need reliable in-process events (`message.received`, `message.extracted`, `action.dispatch`) without introducing Kafka/RabbitMQ at current scale.

## Decision
Use a **transactional outbox table** written in the same DB transaction as domain writes, drained by an Arq worker that publishes to consumers (n8n, internal handlers). Use `LISTEN/NOTIFY` for low-latency wake-ups.

## Consequences
+ Exactly-once semantics from the DB's perspective; no message loss.
+ Cheap: no new infrastructure.
- Not suitable beyond ~a few thousand events/sec; revisit at that threshold and consider NATS/Kafka.
