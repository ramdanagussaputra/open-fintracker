# Proposed App Architecture: Agentic Financial Tracker

> Synthesized 2026-10-30 (Asia/Jakarta). **Research and proposed design only.** The repository contains documentation, with no application, infrastructure, migrations, tests or deployment to validate. All paths, interfaces, controls and runtime nodes below are proposed. The three input notes remain unchanged.

## Recommendation and source precedence

Build a TypeScript modular monolith: **one Next.js/React PWA and authenticated request API, plus one separate Inngest Connect worker using the same domain modules**. Use self-hosted PostgreSQL, Better Auth, private filesystem receipts, self-hosted Inngest with SQLite and persisted Redis, and AI SDK through 9router. Manage Mac and AWS containers with Terraform; add an AWS infrastructure root when deployment is funded.

This is a synthesis of [product requirements](./agentic-financial-tracker.md), [module specification](./architecture-and-system-design.md), and the newer [tech-stack proposal](./tech-stack.md). Product behaviour and domain vocabulary come from the first two; the latest tech-stack note takes precedence on infrastructure choices. These notes record user decisions; this synthesis does not promote their remaining proposals into adopted decisions.

| Topic | Current basis | Resolution in this proposal |
|---|---|---|
| Product | Single-owner MVP, PWA/offline capture, IDR, double-entry, configurable payday periods, 9router, `propose_only`, receipt deletion seven days after approval | Preserve these requirements. Plan for small multi-user growth separately. |
| Supabase and hosted orchestration | Latest note records removing Supabase and self-hosting Inngest as confirmed | Replace Supabase database/auth/storage; remove keepalive work and hosted quota assumptions. |
| Mac/AWS/Terraform | Targets and Terraform ownership confirmed in latest note | Shared runtime module; isolated environments/states. Mac development and EC2 deployment roles remain assumptions. |
| Request API | Latest note recommends Next.js; separate Fastify remains an alternative | Keep HTTP inside Next.js initially. Extract only for measured capacity, protocol or independent-release requirements. |
| Auth and receipts | Better Auth/Google, stable owner mapping and filesystem adapter are proposals | Carry them forward for review, with explicit authorization and expiry checks. |
| Costs | Earlier $0 objective conflicts with permanent AWS hosting | No permanent $0 AWS guarantee. Use existing Mac while strict $0 remains binding; agree AWS ceiling before provisioning. |
| Versions | Earlier notes include historical exact versions | Select compatible supported releases, pin dependencies/images/providers and commit lockfiles during implementation. No compatibility claim here. |

Next.js documents Route Handlers as HTTP endpoints and supports a backend-for-frontend API layer. PostgreSQL, auth and durable jobs provide the other responsibilities in this design; choosing Next.js does not establish those guarantees by itself. [Next.js backend guide](https://nextjs.org/docs/app/guides/backend-for-frontend).

## Runtime component map

[Open the interactive architecture diagram](./diagrams/app-architecture.html). Its [delivery checks](./diagrams/app-architecture.delivery.json), [visual checks](./diagrams/app-architecture.visual-check.json) and [review record](./diagrams/app-architecture.review.json) accompany it.

The following **11 nodes** match the diagram. **Legend:** PWA, web, worker, Inngest and router boxes denote runtime applications; database/file shapes denote persistence. Better Auth is a logical integration inside the web deployment, not another process. SQLite and Redis share one diagram box for readability but remain distinct stores. Internal domain modules are code boundaries named in the diagram cards, not additional deployables. Tailnet HTTPS is an edge; Terraform is the infrastructure control plane outside request flows.

| ID | Diagram node | Responsibility and placement |
|---|---|---|
| `capture` | Tracker PWA | React UI and service-worker app shell on the phone/desktop. |
| `offline` | IndexedDB | Owner-scoped durable command queue and separate selected read snapshots. |
| `web` | Next.js Node API | UI rendering, authorized commands/reads, receipt upload/download and durable dispatch records. |
| `auth` | Better Auth | In-process session integration and provisioned Google subject → owner mapping. |
| `ledger-db` | PostgreSQL | Authoritative ledger, proposals, planning, attachment metadata, auth sessions, audit and dispatch/idempotency records. |
| `receipts` | Private receipt files | Opaque files outside the webroot, shared by web and worker on one host. |
| `jobs` | Self-hosted Inngest | Event intake, schedules, retries, workflow coordination and administration. |
| `job-state` | SQLite + Redis | Separate orchestration history/configuration and persisted queue/run state; no additional application cache. |
| `worker` | Connect worker | Persistent outbound connection to Inngest; domain work, dispatch/reconciliation and AI adapter. |
| `router` | 9router | Private authenticated model endpoint, task Combos and credentials/configuration in its own persistent store. |
| `models` | Allowed models | Selected cloud APIs or optional local Ollama; inexpensive EC2 local inference is not assumed. |

Principal connections: PWA → web over private tailnet HTTPS; web/worker → PostgreSQL and private receipts; web → Inngest publishes IDs; worker → Inngest uses outbound WebSocket; worker → 9router → allowed models; Inngest → its stores. Authentication additionally uses Google's OAuth/OIDC endpoint, an external dependency. The diagram's **Publish IDs** edge means dispatch from a durable application record, retried by a supervised dispatcher; topology alone does not guarantee delivery. PostgreSQL remains the recovery source if sending fails.

Tailscale Serve provides private tailnet sharing with HTTPS. Expose only web ingress; keep orchestration, Redis, PostgreSQL, 9router and optional Ollama on a private container network or loopback. An administrative dashboard requires a deliberately restricted administration path. [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve).

Connect uses an outbound persistent connection and requires a long-running runtime; serverless functions are unsupported. Configure the worker explicitly for the self-hosted gateway, rather than deploying an `/api/inngest` callback route. [Connect requirements and self-hosting](https://www.inngest.com/docs/durable-execution/deploying-functions/connect).

## Domain ownership and code boundaries

Keep one financial domain context initially and seven module seams. Follow the repository's [domain documentation rules](../agents/domain.md) when recording adopted terminology and ADRs. These module names retain the existing specification's vocabulary.

| Proposed module | Owns and guarantees | Public operations / dependencies |
|---|---|---|
| `ledger` | Account, Category, Transaction, Posting, Transfer, AuditLog, exact Money; signs, balancing, valid references, replay and atomic audit | `commitEntry`, `reverseEntry`, balances/activity; PostgreSQL. |
| `proposals` | AgentProposal/AutonomyMode, semantic duplicate review, decision state transitions | Submit, decide, undo; composes ledger and retention in one database transaction. |
| `intake` | Receipt interpretation, deterministic IDR parsing, extraction validation and arithmetic checks | Extract/classify through `AiGatewayPort`; no authority to post money. |
| `planning` | Budget, BudgetPeriod, RecurringRule, Occurrence, Goal | Resolve periods, materialize occurrences, read budget/goal progress; ledger for posting. |
| `insights` | InsightReport and deterministic metrics, optional grounded narrative | Read ledger/planning, narrate through AI port, retain deterministic fallback. |
| `retention` | Attachment metadata, file lifecycle, image hash, approval expiry and purge recovery | Store/read/mark-approved/purge through `BlobStoragePort`. |
| `offline-outbox` | Browser pending commands and receipt captures | Enqueue/replay/status through `OutboxTransportPort`; IndexedDB. |

Cross-module callers import `src/modules/<name>/index.ts`, never another module's internals. Domain modules do not import Next.js, Inngest or concrete adapters. Auth/session resolution and HTTP parsing belong to request adapters; scheduling and workflow steps belong to worker adapters. Application composition supplies the verified owner, transaction and ports. Keep framework entry points thin without treating an arbitrary line limit as correctness.

Use concrete Drizzle database/transaction handles internally rather than adding pass-through repositories. Keep explicit AI, blob-storage and browser-transport ports with production and deterministic/in-memory test adapters. PGlite and fake-indexeddb support fast module tests; real PostgreSQL/browser tests remain necessary for roles, concurrency, filesystem and browser behaviour. [Drizzle transactions](https://orm.drizzle.team/docs/transactions), [PGlite](https://pglite.dev/docs/), [fake-indexeddb](https://github.com/dumbmatter/fakeIndexedDB).

Proposed organization: `src/app` for HTTP/UI, `src/modules` for domain seams, `src/adapters` for external implementations, `src/worker` for Connect/composition, `src/db` plus `drizzle/` for schema/migrations, and `infra/modules/runtime` with separate environment roots. Replace the old Supabase adapter with `LocalFilesystemBlobStorageAdapter`. Keep offline wire schemas shared without importing server-only runtime code into the browser.

## Main flows and failure handling

### Manual or offline financial command

1. UI validates a capture and saves its versioned command, stable client ID and owner UUID into IndexedDB before claiming it is queued. Cache snapshots separately and label their sync time.
2. Replay through a stable HTTP Route Handler on startup, foreground return, online events or explicit retry. Background Sync is an enhancement because browser support is limited and requires a secure context. [Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API).
3. Web verifies session/owner, checks schema and references, and calls ledger. Commit Transaction, Postings, AuditEvent and owner-scoped idempotency result atomically. The same key plus a different command payload is a conflict; the same command returns its original outcome.
4. Remove the browser command only after durable acknowledgement. A lost response is safe to retry. Authentication expiry pauses replay; validation errors remain editable and visible. After logout/account change, clear financial read caches and explicitly handle unsynced captures; never replay under another identity.

IndexedDB is subject to quotas and eviction. Request persistent storage where supported, cap pending image count/size and surface write failures; installation or persistence requests do not establish an absolute no-loss guarantee. [Browser storage quotas](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria).

### Receipt → proposal → approved ledger change

1. Preprocess the image, strip metadata and retain a bounded upload locally until acknowledged. Web verifies type/size, computes the hash itself and stores an opaque private file.
2. Persist Attachment plus a durable dispatch record in PostgreSQL. File I/O and SQL are not one transaction: use temporary files/atomic rename, recorded status and orphan/missing-file reconciliation. Acknowledge only a durable recoverable upload state.
3. Dispatch `attachment.uploaded` with record IDs to self-hosted Inngest. A failed send leaves the dispatch pending. A supervised dispatcher retries pending records; startup/periodic reconciliation finds stalled captures. Delivery and processing can repeat, so database uniqueness is still required. **This persistent dispatch record is a proposed refinement addressing the earlier direct DB-write/event-send gap.**
4. Connect reads the authorized Attachment, invokes 9router through the AI adapter, validates extraction and persists an idempotent AgentProposal. Return only IDs/status from durable steps; fetch bytes and perform extraction/persistence within a step rather than checkpointing an image-returning step. Inngest persists completed step results, which would otherwise create extra image/extraction copies. [Durable workflow checkpoints](https://www.inngest.com/docs/durable-execution/durable-workflows).
5. UI shows proposed fields, unresolved references, duplicate candidates and validation warnings. Owner approves, edits or rejects through an authenticated command. Approval locks/checks the proposal version, revalidates references/amounts and atomically commits postings/audit, proposal result and `approved_at`/expiry. Competing decisions resolve once; rejection posts no money.
6. Undo uses ledger reversal/void semantics and append-only audit; preserve the original Transaction and prevent repeated reversal. Do not reset the original image expiry after edit, retry or undo.

Hash deduplication is exact for identical uploaded bytes; recompression/cropping can change hashes. Semantic matches require review and must not silently merge two distinct purchases. A receipt plus payment screenshot may link to one existing Transaction after confirmation, rather than produce another expense.

### Scheduled work, reports and retention

Run recurrence maintenance, weekly reports at Monday 08:00 WIB, budget-period reports and attachment cleanup through self-hosted Inngest. Store RRULE, DTSTART and `Asia/Jakarta` separately. Unique rule/date keys make materialization repeatable. RFC 5545 skips invalid dates; `BYMONTHDAY=31` does not mean “last day of every month.” Use an explicit last-day rule and show its meaning. [RFC 5545 §3.3.10](https://www.rfc-editor.org/rfc/rfc5545#section-3.3.10).

Compute balances, spend, pace, goal math and report metrics deterministically. Let the LLM select metric references and narrative structure; resolve display values from those metrics. A numeric whitelist alone cannot prove a claim's meaning or attribution. Render validated references/templates, fall back to deterministic text, and retain the metric snapshot/report period for reproducibility. This tightens the earlier prose-number check without claiming perfect factual grounding.

Set `retention_expires_at = approved_at + 7 days` in the approval transaction and deny preview/download at expiry. Schedule expiry work plus periodic/startup reconciliation; retry deletion safely and mark completion only after file removal. Keep dedup metadata and structured financial records. A daily purge can lag by almost a day, and a sleeping host cannot physically delete files: always-on execution and an agreed deletion-delay bound are release decisions, not guarantees already provided.

Exclude receipt bytes from workflow history, debug logs, long-lived backups and query/service-worker caches. Delete acknowledged browser upload copies and temporary files. If image backups are required, their copies must expire by the original deadline; restore must reapply expiry before serving files. Decide rejected/abandoned attachment expiry separately.

## Storage, money and authorization

PostgreSQL is the financial source of truth. Use exact integer minor units with currency, positive command amounts and signed Postings. Preserve the existing IDR convention of 100 minor units per rupiah; transmit integer decimal strings in JSON and format rupiah for UI. Parse Indonesian separators/shorthand deterministically without floating-point amount arithmetic. Separate constructors for positive movement amounts and nonnegative balances/fees so a zero opening balance is valid. Preserve currency columns; FX semantics remain future design work, not an automatic result of adding a rate column. [PostgreSQL numeric types](https://www.postgresql.org/docs/current/datatype-numeric.html).

Each posted Transaction has at least two Postings and sums to zero per currency. A Rp100.000 transfer plus Rp1.000 fee debits the source Rp101.000, credits destination Rp100.000 and records Rp1.000 expense; principal does not consume Budget. Foreign keys, row checks and owner-scoped uniqueness enforce local invariants. Review a deferred constraint trigger or controlled write procedure for cross-row balancing; a normal CHECK cannot enforce sums over other rows. [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html), [constraint triggers](https://www.postgresql.org/docs/current/sql-createtrigger.html).

Keep financial ownership independent of auth implementation: provision a stable application owner UUID and map the allowed Google subject to Better Auth identity and then that UUID. Never let an arbitrary first login claim ownership or accept owner IDs from request bodies as authority. Google documents `sub` as stable while email can change. Better Auth supports Next.js session lookup, but a successful sign-in does not grant this app's financial access. [Google OIDC](https://developers.google.com/identity/openid-connect/openid-connect), [Better Auth integration](https://better-auth.com/docs/integrations/next).

Use scoped auth-table access, separate migration credentials and a non-owner financial runtime role without superuser/BYPASSRLS. Set verified owner context transaction-locally on the same pooled connection; financial RLS must deny missing/wrong context. Jobs validate ownership from authoritative records. Table owners normally bypass RLS, so role design and real database tests matter. RLS guards query mistakes; application-controlled context does not withstand a fully compromised application. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

Receipt routes authorize owner and expiry on every read, send private/no-store responses and resolve generated keys within a restricted directory. Prevent traversal/symlink escape, enforce MIME/magic-byte/upload limits and never expose a static receipt directory. Filesystem storage does not provide authorization by itself. [OWASP upload guidance](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html).

## Agent authority and model boundary

The MVP agent produces pending AgentProposals only. Receipt extraction has **no tools**. Later chat read tools may expose scoped accounts, categories, ledger search, budgets and goal progress; its only write tools are `propose_transaction` and `propose_transfer`. No model gets database credentials, arbitrary SQL, ledger commit/reversal, approval, shell execution or arbitrary-network tools. Approval authority belongs to the authenticated decision endpoint and deterministic domain transition, persisted in PostgreSQL rather than an in-memory SDK approval conversation.

Treat receipt text and model output as untrusted data. Zod/schema checks establish shape, then deterministic code checks money, references, currency, dates, arithmetic and duplicates. Arithmetic agreement is not evidence that OCR read the right number. Keep failed/uncertain extraction visible for manual correction; never manufacture a successful posting after provider failure.

Use `AiGatewayPort` with AI SDK's OpenAI-compatible adapter to private 9router. Send bounded inline image bytes, keep credentials server-side and disable optional token-saving request rewriting with the router's documented header where supported. Pin and contract-test vision, structured JSON, fallback, timeouts and returned model identity for the actual release/Combo; OpenAI compatibility does not establish identical capabilities for every target. [AI SDK compatible provider](https://ai-sdk.dev/providers/openai-compatible-providers), [9router primary repository](https://github.com/decolua/9router).

Provider privacy/cost eligibility and task/model mapping remain decisions. Permit only selected targets; no silent paid fallback. A local-only Combo must disable cloud fallback. Router availability does not grant provider entitlement or unlimited free inference. Keep manual entry usable during AI outages. Do not describe `min(model confidence, deterministic score)` as calibrated probability; later automation needs a representative labelled evaluation set, bounded amount/category policy, duplicate checks, explicit opt-in and reversal support. Earlier “about 50 approvals” is a product heuristic, not a demonstrated safety threshold.

## Proposed stack and deployment

| Area | Recommendation |
|---|---|
| Runtime/UI | Supported Node.js LTS satisfying Connect; Next.js App Router/React; Tailwind/shadcn, optional Recharts and Motion with reduced-motion support. |
| Validation/data | TypeScript strict mode, Zod; PostgreSQL + Drizzle/pg and reviewed SQL migrations. |
| Browser persistence | `idb` for indexed command records; TanStack Query persistence only for selected owner-scoped read snapshots. |
| Identity/files | Better Auth + Google OAuth; private local filesystem adapter. |
| Jobs | Self-hosted Inngest production server, separate Connect worker, SQLite volume and dedicated persisted Redis. |
| AI | AI SDK + OpenAI-compatible adapter → 9router; optional local Ollama when suitable hardware exists. |
| Verification | Vitest/PGlite/fake-indexeddb; real PostgreSQL integration checks and Playwright plus actual iOS PWA checks. |
| Operations | Terraform AWS + Docker providers, persistent volumes, private Tailscale HTTPS, restricted off-host backups. |

Inngest's documented self-hosting supports its single binary and external persistence options. Retain the latest note's **SQLite plus external Redis** proposal; application PostgreSQL does not silently become its orchestration database. Redis AOF `everysec` plus snapshots is a proposed tradeoff with possible recent-write loss, not zero-loss execution. Restore both orchestration stores coherently and reconcile from application records. [Inngest self-hosting](https://www.inngest.com/docs/self-hosting), [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/).

Use Terraform roots `mac-runtime`, `aws-infra`, `aws-runtime` with separate state and one shared container module. Docker engine installation/readiness is a prerequisite on Mac; EC2 bootstrap mounts durable encrypted EBS before the runtime apply. Terraform manages declared resources; restart supervision and application retries handle ongoing execution. Do not expose the Docker API publicly. [Docker provider documentation](https://github.com/kreuzwerker/terraform-provider-docker/blob/master/docs/index.md).

Keep persistent PostgreSQL/receipts/Inngest/Redis/9router data outside disposable container layers. Fail startup if required mounts are absent. Protect persistent resources against accidental planned deletion, review replacements, and restore-test backups; Terraform state is not a financial backup. Keep secrets outside Git and preferably in restricted runtime-mounted files. `sensitive` redacts display but does not inherently omit values from Terraform state. [Terraform sensitive data](https://developer.hashicorp.com/terraform/language/manage-sensitive-data).

Mac and AWS are independent environments; no automatic synchronization or HA is proposed. A host outage pauses requests/jobs; a previously installed cached PWA can capture pending commands. Reconcile missed recurrence, dispatch, expired attachments and absent reports after restart rather than assuming cron replay. EC2 compute/storage/networking and backups can cost money; no AWS resources or paid inference are authorized by this research. [EC2 pricing](https://aws.amazon.com/ec2/pricing/on-demand/).

## Phased implementation and readiness gates

| Phase | Deliverable | Evidence required before moving on |
|---|---|---|
| 0 — settle operating contract | Adopt decisions/ADRs, auth owner, retention deadline, Mac/AWS role and budget | Review open decisions below; choose compatible pinned versions and backup/recovery targets. |
| 1 — ledger vertical slice | Owner-only login; Account/Category; manual expense/income/transfer; migrations and audit | Hand-computed fee example, invalid amounts, zero balances, reversal, concurrent replay, cross-owner/RLS and pooled-context checks on real PostgreSQL. |
| 2 — offline commands | Installable PWA, separate read cache and versioned durable IndexedDB queue | Offline save/replay, lost response, auth expiry, logout identity change, quota failure and old-command replay after deployment; actual iPhone foreground recovery. |
| 3 — assisted receipt slice | Private files, dispatch/reconciliation, Connect/9router adapter, pending proposals and explicit approval | Crash between file/DB/event stages, duplicate upload, competing decisions, no unapproved posting, representative receipt accuracy and provider-failure recovery. |
| 4 — planning and operations | Budget periods, recurrence, goals, deterministic reports, expiry/recovery jobs | Short months/payday edges, repeated/missed schedules, grounded fallback, overdue purge alerts, backup restoration and absence of receipt copies in history/logs. |
| 5 — deployment | Terraform-managed Mac runtime, then funded AWS environment | fmt/validate/plan review, convergence, image replacement with data retained, missing-mount failure, restricted ingress/secrets, restore drill and agreed resource/latency budgets. |
| 6 — small multi-user evolution | Controlled admission and independent owners, unless workspaces are deliberately selected | Tenant isolation for HTTP/jobs/files/outbox; measured load/pool budget and per-user AI admission; deployment/backup targets updated. |

Start one host and one PostgreSQL writer. Bound upload/pending work, global/provider concurrency and per-user AI usage. Observe p95 request latency/errors, pool waits/locks, host resources, queue age, worker connectivity, expiry lag and backup age. Set numeric targets from the agreed workload, then measure; no capacity or availability score is established here.

Before adding web/worker hosts, replace local receipt volumes with selected shared/private object storage behind the same port, coordinate Next.js builds/cache/Actions and database pool budgets, and review orchestration persistence/HA independently. More Connect workers do not make a single SQLite/Redis orchestration host highly available. Keep a separate Fastify API, Kubernetes, a vector database, full offline database replication and extra agent frameworks deferred until a concrete requirement justifies their operation.

## Open decisions

1. **Operating and cost contract:** Mac-only production versus Mac development/EC2 production; AWS region/size/storage/monthly ceiling; availability and recovery targets. Permanent strict $0 and always-on AWS cannot both be assumed.
2. **Retention:** acceptable physical deletion delay, always-on execution, rejected/abandoned image expiry, browser/temp/backup copies, and retention for raw model outputs/audit/report data. Read denial at seven days alone does not satisfy physical deletion.
3. **Identity:** adopt Better Auth/Google and provisioned subject mapping; validate the real private HTTPS OAuth callback and owner recovery path.
4. **Inference:** permitted model/provider terms and data use, local-only privacy mode, resource suitability and per-task accuracy/cost/timeout limits. No upstream entitlement or accuracy has been tested.
5. **Financial semantics:** confirm reversal/void/edit representation and opening-balance audit policy; IDR input/rounding edge cases; recurrence auto-post is an explicit user-created rule permission, distinct from agent autonomy.
6. **Growth:** independent personal owners versus shared households/workspaces, admission, tailnet eligibility/reachability, expected workload and privacy obligations before admitting others. User-ID columns alone do not make a multi-user app ready.
7. **Persistence operations:** adopt SQLite+Redis orchestration, history cleanup, coordinated backup/restore and dispatch ownership; select state/backup destinations and deployment maintenance windows.

The resulting design keeps financial authority in deterministic domain modules and PostgreSQL, while the browser handles capture, Inngest handles durable coordination, and 9router handles selected inference. It remains a reviewable implementation proposal, with its gates explicitly unverified.
