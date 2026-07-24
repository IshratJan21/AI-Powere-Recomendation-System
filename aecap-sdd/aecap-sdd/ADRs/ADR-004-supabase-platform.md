# ADR-004: Supabase for DB, Auth, Storage

**Status:** Proposed
**Date:** 2026-07-24

## Context
We need Postgres, authentication, object storage, realtime, and secrets — with a small team.

## Decision
Adopt **Supabase** as the platform layer: Postgres (with pgvector), Auth (JWT/RS256), Storage, Realtime, and Vault for secrets. Multi-tenancy enforced by **Row-Level Security**.

## Consequences
+ One provider, integrated JWT with the Next.js frontend, native RLS multi-tenancy.
+ Vault removes the need for a separate KMS.
- Vendor concentration risk — mitigated by using standard Postgres/JWT (portable).
