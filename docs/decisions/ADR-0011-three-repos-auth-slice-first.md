# ADR-0011: Split into three repositories, starting with the authentication slice

> Related: [AGENTS.md](../AGENTS.md) · [master-vision.md](../master-vision.md) · [project-map.md](../project-map.md) · [api-integration.md](../api-integration.md)

- **Status:** Accepted
- **Date:** 2026-07-16
- **Deciders:** Sponsor (CISO/DPO), PM/Legal owner, Architecture
- **Supersedes / Superseded by:** —

---

## Context

Al-Mustashar is a Yemeni legal AI advisor that must run **fully On-Prem inside Yemen**, with strict data-sovereignty, privacy, and security-isolation requirements.

The system is made of three concerns that have fundamentally different lifecycles, skill sets, deployment targets, and security boundaries:

1. **The Flutter mobile client** — user-facing UI, ships to app stores / devices.
2. **The FastAPI backend** — API + PostgreSQL (pgvector) + inference; holds all personal and legal data; runs On-Prem.
3. **Governance / compliance** — GRC specifications, ISO/IEC 27001 controls, OWASP LLM guidance, and legal/policy artifacts.

Additional forces:

- The backend follows a **contract-first, hexagonal, modular-monolith** design and must be the **single owner of the API contract**.
- We want to ship **incrementally** and prove the architecture end-to-end before widening scope.
- Both human developers and AI coding agents work faster with **small, focused repositories** that fit a clean context window.

## Decision

Use **three separate Git repositories**:

| Repository | Responsibility | Own `docs/` set |
|---|---|---|
| `almustashar` | Flutter mobile app (client) | `frontend-AGENTS.md`, `project-map.md`, `master-vision.md`, `MEMORY.md` |
| `almustashar-api` | This backend — FastAPI, PostgreSQL + pgvector, inference adapters. **Owns the API contract (`openapi.json`)**. Local folder: `almustashar_backend`. | full backend docs set (batches 1–3) |
| `almustashar-grc` | Governance, risk & compliance specs and legal/policy artifacts | governance docs |

Principles:

- **Each repository is self-contained** with its own documentation set.
- **Shared understanding is synchronized through the API contract (`openapi.json`)** and the human-facing Single Source of Truth in Notion — **never** by duplicating code or design docs across repos. (e.g. frontend UI rules stay in the frontend repo; the backend owns the integration contract in `api-integration.md`.)
- **Delivery starts with the authentication slice (Phase أ)**, implemented **vertically through all backend layers first**, before any other feature — to validate the hexagonal architecture, fail-closed RLS, the security model, and the BE↔FE contract end-to-end.

## Consequences

### Positive
- Clear separation of concerns and ownership.
- Independent **CI/CD and versioning** per repository.
- A hard **security boundary**: the On-Prem backend + data stay isolated from the client.
- **Contract-first integration** via a single, owned `openapi.json`.
- Smaller, focused repos are easier for humans **and AI agents** to reason about.
- The **auth-first vertical slice** de-risks the architecture early.

### Negative / costs
- The API contract must be kept **in sync** across the backend (producer) and the frontend (consumer).
- Cross-repo changes require **coordination**.
- Each repo maintains a **parallel `docs/` structure**.
- No single-checkout "build everything at once".

### Follow-ups / mitigations
- **Freeze and version `openapi.json`** as the official contract (M1) and **delete the broken `openapi.yaml`**.
- Keep each repo's `docs/MEMORY.md` and the Notion SSOT **consistent**.
- Establish **CI in each repository**.
- Document the client integration contract in `docs/api-integration.md` (done).

## Alternatives considered

1. **Monorepo (all three concerns in one repository)** — *Rejected.* Blurs the security boundary required for On-Prem isolation, couples deployments and release cadence, and makes the repo harder to fit into a focused AI-agent context window.
2. **Single combined app + backend repository (GRC separate)** — *Rejected.* Still couples the client and server lifecycles/deployments and weakens the clean contract-first producer/consumer boundary.

## Links

- Docs: `docs/master-vision.md` (three-repo overview, phase roadmap), `docs/project-map.md`, `docs/api-integration.md`, `docs/AGENTS.md`
- Milestone: **M1** — stabilize + first real BE↔FE auth link
- Notion: architecture / decisions section of the project SSOT
