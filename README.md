# Proto draft — the domain model

Status: **DRAFT**. Not ratified. Companion to
`../architecture-decisions-2026-08-29.md`; every non-obvious choice here cites the
decision it comes from.

Under D16 these `.proto` files are the contract shared across module repos —
the artifact, not a Rust crate. `buf breaking` gates them in CI so D15's
additive-only rule is enforced structurally rather than by discipline.

## What is here

| Path | Contents |
|---|---|
| `yadgar/common/v1` | `Meta`, `Visibility`, `Scope`, `Idempotency` — imported by everything |
| `yadgar/memory/v1` | `Memory` + `MemoryDbService` — ranked, implements the provider contract |
| `yadgar/wiki/v1` | `WikiPage` + `WikiDbService` — ranked |
| `yadgar/adr/v1` | `Adr` + `AdrDbService` — addressed, deliberately no provider |
| `yadgar/task/v1` | `Task` + `TaskDbService` — addressed |
| `yadgar/recall/v1` | `RetrievalProviderService` fanout contract + `RecallService` |
| `yadgar/audit/v1` | `WriteEvent` / `ReadEvent` + `AuditDbService` |

`AgentPrompt`, `Block`, and `Bookmark` follow `yadgar/adr/v1` exactly (addressed,
no provider). `Entity` / `Relationship` follow `yadgar/memory/v1` (ranked,
graph-scored). They are omitted only to keep the draft reviewable.

Verified with `buf lint` and `buf build` (buf 1.72.0) — both clean, re-run after the
D39–D46 amendments. The directory
layout mirrors the package path and services carry the `Service` suffix because
buf's `STANDARD` lint set requires both; `Db` is kept in the name to say which
half of a logic/`-db` twin pair the service is.

## Conventions

- **Envelope in 1–15.** Those field numbers encode in one byte and appear on
  every row of every store. 13–15 are reserved for envelope growth; entity
  fields start at 16.
- **Every mutating rpc carries `Idempotency`** (D9) — a retrying load balancer
  will deliver a write more than once. A repeated key is a **replay**: it returns the
  original outcome and is never reported as a failure, or an instance told its write
  failed will "fix" it by writing a duplicate.
- **Every update carries `expect_version`** (D8). Mismatch is
  `FAILED_PRECONDITION`; the caller re-reads, recomputes, retries.
- **`Scope` is attested, never self-declared.** The gateway derives it from the
  request's credentials. A caller cannot claim a `team_id` it is not in.
- **Batch reads exist on every module** so a fanout never becomes an N+1 across
  the network — the specific risk D5's coarse-API rule exists to prevent.
- **One delete verb**, owner-only, soft (D26). No unshare rpc, no erase rpc.
  Physical removal is administrative.
- **Services are coarse:** one rpc = one business operation = one transaction
  (D5). No generic query rpc, no exposed `begin`/`commit`.

## The two load-bearing pieces

**`recall/v1` is what makes "no shared substrate" work.** Each searchable module
implements `RetrievalProvider` over its own store; the orchestrator fans out and
reranks the union. Two details carry the design:

- the query embedding is computed **once** and passed to every provider, so N
  providers do not each re-embed the same text;
- providers return a `snippet`, because making the orchestrator fetch text for
  reranking would reintroduce the N+1 the batch rpcs exist to avoid.

Provider scores are explicitly **not** comparable across providers. Comparability
comes from the cross-encoder scoring the union uniformly. `ScoreKind` is for
diagnosis and weighting, never for ordering across sources.

**`audit/v1` splits by guarantee, not by taste.** Write events go through a
transactional outbox and cannot be lost. Read events are published
fire-and-forget because a read has no transaction to attach to — so a gap in the
read stream is not evidence a read did not happen.

## Open

- ~~**ID scheme.**~~ **RESOLVED by D42.** UUID-in-URN everywhere: UUIDv7 for
  generated records, UUIDv5 over namespace + source path for seeded ones, so
  re-seeding updates in place rather than duplicating (D35). `Adr.number` and
  `Task.number` stay as separate integer sequences — they are scoped human
  references, not identity. Names carry no uniqueness anywhere.
- **`provenance_agent`** sits on `Memory` only. Whether it belongs in `Meta` is
  undecided — see D12.
- **Reverse crossrefs** (O4). `WikiPage.links` holds outbound edges only, so
  "what links here" is a fanout until a decision is made.
- **Not yet drafted, and now required.** `viz/v1` (`Dashboard`, D44), `profile/v1`
  (namespaced preference blobs, D43), and the D39 notice channel — the shape that
  carries a failed write back to an instance on its next call, which every module's
  write path depends on. `config/v1` shrinks to reads only (D43).
- **Pagination** uses page-token strings; the token format is unspecified.
