# AECAP — AI Email & Chat Automation Platform

This ZIP contains the **Software Design Document (SDD)** for AECAP, plus supporting artifacts.

## Contents

```
aecap-sdd/
├── README.md                              # This file
├── docs/
│   └── Software_Design_Document.md        # Full SDD (20 sections)
├── ADRs/
│   ├── ADR-001-modular-monolith.md
│   ├── ADR-002-fastapi-backend.md
│   ├── ADR-003-n8n-orchestration.md
│   ├── ADR-004-supabase-platform.md
│   ├── ADR-005-ai-provider-abstraction.md
│   ├── ADR-006-event-bus.md
│   └── ADR-007-structured-output-contract.md
├── schemas/
│   └── extraction_envelope.schema.json    # Canonical AI output contract
├── diagrams/
│   ├── system_architecture.txt
│   ├── workflow.txt
│   ├── ai_processing_flow.txt
│   └── er_diagram.txt
└── folder_structure.txt                   # Proposed monorepo layout
```

## Status
**Draft — awaiting architecture approval.** No implementation code will be produced until sign-off.

## Approval Sequence
1. Principal Engineer sign-off
2. Security review
3. SRE review
4. Product review
5. Legal / Compliance review

Once approved, implementation proceeds in this order:
1. Repo scaffold + CI
2. Supabase schema + RLS
3. FastAPI skeleton with ports
4. AI provider abstraction with contract tests
5. Gmail ingestion adapter (vertical slice)
6. First n8n workflow
7. Next.js inbox MVP
