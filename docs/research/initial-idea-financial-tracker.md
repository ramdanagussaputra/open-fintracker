# Initial Idea: Financial Tracker

> Summary and navigation for all six research documents in this directory, assembled 2026-11-01 (Asia/Jakarta). This is a proposed product and implementation direction. The research does not establish a working application, tested deployment or adopted coding standard.

## The idea

Build a personal financial tracker that makes daily money capture easy while keeping the financial record reliable. The owner records an expense, income or transfer manually, or photographs a receipt/payment screenshot and reviews the AI's suggested transaction. Budgets follow the owner's payday cycle; goals and periodic reports show where the money went and whether saving is on track.

The first version serves one owner, uses Indonesian rupiah (IDR), and works as an installable mobile-first PWA. Offline capture matters: the app saves pending commands and receipt captures locally, then submits them when the server becomes reachable. It shows cached balances with their last sync time and distinguishes locally saved work from a posted transaction.

AI assists with interpretation and explanation. Deterministic code owns amounts, balances, authorization and financial changes; the MVP uses `propose_only`, so receipt extraction produces a pending `AgentProposal` for explicit approval. See the [product requirements](./agentic-financial-tracker.md#2-functional-requirements) and [current agent authority proposal](./app-architecture.md#agent-authority-and-model-boundary).

## Research map and reading order

Read the overview here first, then choose the document that answers the next question. The order below moves from product intent to implementation detail; the precedence rules in the next section resolve conflicting recommendations.

| Order | Research document | Summary and when to read it |
|---|---|---|
| 1 | [Agentic Personal Financial Tracker](./agentic-financial-tracker.md) | Original product discovery: features, financial entities, IDR handling, receipt capture, model routing, offline UX and privacy concerns. Start here for **what the app should do**. Its Supabase/hosted-service picks, package versions and free-tier assumptions are historical. |
| 2 | [Proposed Tech Stack](./tech-stack.md) | Updated operating direction after removing Supabase and choosing self-hosted Inngest. Compares Next.js with a separate API, proposes replacement services, defines Terraform ownership on Mac/AWS, and examines cost, recovery and small multi-user growth. Read for **what could run the app and what operating it requires**. |
| 3 | [Proposed App Architecture](./app-architecture.md) | Connects the product and stack into runtime components and end-to-end flows. Adds durable dispatch, file/SQL recovery, approval concurrency, authorization, retention and phased readiness gates. Read for **how the parts cooperate and recover from failure**; its Drizzle-in-core placement yields to the newer structure note. |
| 4 | [Architecture, System Design & Deep-Module Specification](./architecture-and-system-design.md) | Detailed original module catalogue, vocabulary, TypeScript interface sketches, sequence diagrams, transfer examples and test scenarios. Read for **the reasoning behind the behavioural interfaces**. Apply the later corrections before using its snippets or topology. |
| 5 | [Proposed Codebase Structure](./codebase-structure.md) | Latest source organization proposal: feature-first modules, a pure `money` module, core-owned persistence/atomic-work contracts, adjacent SQL adapters and separate browser/web/worker composition. Includes a named file tree and adapter verification strategy. Read for **where code belongs and which imports are allowed**. |
| 6 | [Coding Standards](./coding-standards.md) | Turns the design into a proposed contributor contract: readable names/control flow, strict TypeScript, validation, module ownership, exact values, async/error handling, UI states, security, tests and delivery gates. Read for **how to implement and review changes**. Its scripts and checks are targets for adoption, not existing enforcement. |

For a visual runtime map, open the [interactive app architecture diagram](./diagrams/app-architecture.html). It accompanies the app-architecture proposal; use the latest structure note for source dependencies.

## How to read conflicting recommendations

The notes record an evolving design. Preserve the product requirements while applying explicit later corrections within their scope.

| Topic | Basis for this summary | Consequence |
|---|---|---|
| Product behaviour | [Original requirements](./agentic-financial-tracker.md#2-functional-requirements), refined by [app flows](./app-architecture.md#main-flows-and-failure-handling) | Retain single-owner, IDR, PWA/offline capture, double-entry, payday periods, 9router, pending proposals and seven-day post-approval image retention. |
| Infrastructure | [Tech-stack recommendation](./tech-stack.md#1-recommendation) | Removing Supabase and self-hosting Inngest are confirmed choices recorded in the research. Supabase Auth/Storage, keepalive jobs and hosted Inngest quotas no longer describe the proposed runtime. |
| Source dependencies | [Structure revision](./codebase-structure.md#recommendation-and-evidence) | Core-owned persistence and transaction contracts replace earlier direct Drizzle handles in financial modules. SQL adapters implement those contracts from outside the core. |
| Coding and verification | [Standards coverage and priority](./coding-standards.md#coverage-of-the-existing-research) | Use the newer corrections to parsing, rollback and testing. Avoid copying the older snippets as production-ready code. |
| Cost and versions | [Cost discussion](./tech-stack.md#5-what-0-means-here) and [app source precedence](./app-architecture.md#recommendation-and-source-precedence) | Permanent $0 AWS hosting and unlimited free inference are unproven. Historical exact package versions need compatibility checks and fresh pinning during implementation. |

The research records Mac/AWS targets and Terraform for infrastructure/runtime management as confirmed. Better Auth/Google, filesystem receipts, the Next.js request API, SQLite plus Redis for orchestration, and detailed port signatures remain proposals. This overview does not adopt them by implication.

## First-version experience

| Capability | Intended behaviour |
|---|---|
| Accounts and categories | Track cash, bank accounts, e-wallets and cards; organize income and expense categories. |
| Manual transactions | Add, search and correct expenses, income and transfers through a simple UI backed by balanced postings and an audit trail. |
| Offline capture | Save commands and bounded receipt images in IndexedDB. Replay on startup, foreground return, connectivity recovery or explicit retry; keep pending/error status visible. |
| Receipt assistance | Upload a receipt or payment screenshot, inspect extracted fields and duplicate warnings, then approve, edit or reject the proposal. |
| Budgets and recurrence | Set category limits with configurable `period_start_day` (`1..28` or `-1` for month-end). Schedule recurring income/expenses using local-time rules in `Asia/Jakarta`. |
| Goals and insights | Track target amounts/dates and required saving. Generate deterministic weekly and budget-period metrics, with optional AI narration grounded in those metrics. |

The mobile UI should support quick IDR entry, accessible status messages and reduced motion. AI failure leaves manual capture usable. Currency columns preserve room for future work, but they do not implement multi-currency accounting. See [MVP scope](./agentic-financial-tracker.md#21-mvp-locked-scope) and [offline/UI standards](./coding-standards.md#react-nextjs-and-offline-ui).

## Proposed technical direction

Use one TypeScript modular monolith that produces a Next.js/React PWA and request API, plus a separate long-running Inngest Connect worker sharing the same application modules. Keep receipt processing and scheduled work outside the HTTP request lifetime.

| Responsibility | Proposed approach |
|---|---|
| Web and browser | Next.js App Router, React, Tailwind/shadcn; service-worker app shell, `idb` for the command queue and TanStack Query for selected read snapshots. |
| Financial source of truth | Self-hosted PostgreSQL, with Drizzle/`pg` in outer adapters and reviewed migrations. No browser-to-database access. |
| Identity and receipt files | Better Auth with provisioned Google-subject-to-owner mapping; private filesystem storage behind authorized, expiry-checked routes. |
| Durable jobs | Self-hosted Inngest, a separate Connect worker, and proposed SQLite plus persisted Redis orchestration stores. Independently supervised dispatch/recovery scans authoritative application records. |
| AI | AI SDK through a private 9router endpoint, with explicitly permitted task/model Combos and optional local Ollama. No silent paid fallback. |
| Infrastructure | Shared Terraform Docker runtime module with separate `mac-runtime`, `aws-infra` and `aws-runtime` roots/states. Mac and AWS are independent environments. |
| Verification | Strict TypeScript, formatter/lint/import checks, Vitest and adapter contracts; real PostgreSQL, filesystem and browser checks for guarantees that fakes cannot establish. |

See [dependency choices](./tech-stack.md#proposed-dependency-set), [runtime topology](./tech-stack.md#2-runtime-topology-and-availability), [Terraform ownership](./tech-stack.md#6-terraform-on-mac-and-aws) and [proposed delivery gates](./coding-standards.md#proposed-check-contract).

### Code ownership

Deep Modules means callers express an intent without reconstructing the underlying rules or write order. Clean Architecture keeps core source independent of the infrastructure that executes those rules. Neither requires a class or forwarding interface for every operation.

| Module | Owns |
|---|---|
| `money` | Pure, exact IDR parsing and value semantics shared by browser capture and server modules. |
| `ledger` | Accounts, categories, balanced transactions/postings, audit, reversal, replay and authorized reads. |
| `proposals` | Proposal decisions, duplicate review and atomic approval across ledger and attachment expiry. |
| `intake` | Untrusted extraction validation and interpretation; no approval or ledger-write authority. |
| `planning` | Budget periods, recurring occurrences and goals. |
| `insights` | Deterministic metrics, validated narrative references and fallback reports. |
| `retention` | Attachment metadata, recoverable file lifecycle, fixed approval expiry and purge recovery. |
| `offline-outbox` | Owner-scoped browser commands, local acknowledgement, retry and server acknowledgement. |

Callers use supported module entries. Core-owned ports describe persistence, atomic work and external capabilities; composition connects their concrete adapters. Simple authorized reads may use SQL projections through those contracts. See [module ownership](./codebase-structure.md#ownership-and-interfaces-for-this-app), [atomic work](./codebase-structure.md#atomic-work-without-leaking-drizzle) and [the proposed directory tree](./codebase-structure.md#proposed-directory-tree).

## Rules that protect the financial record

Money uses exact integer minor units with currency, preserving the research convention of 100 minor units per rupiah. Localized input parsing must remain exact; JSON carries integer amounts as decimal strings. Each posted transaction has at least two postings that sum to zero per currency. A transfer's principal does not count as spending: moving Rp100.000 with a Rp1.000 fee decreases the source by Rp101.000, increases the destination by Rp100.000 and consumes Rp1.000 of expense budget.

Financial commands commit their postings, audit and replay result together. Approval also commits the proposal outcome and first-approval receipt expiry in the same transaction. Replaying an acknowledged command returns its original outcome; reusing its ID with different input produces a conflict. Verified owner scope applies to requests, jobs and file reads. See [financial rules](./coding-standards.md#financial-values-dates-and-state-changes) and [transaction semantics](./coding-standards.md#atomicity-is-part-of-the-contract).

For receipts, the flow is: local capture → recoverable private upload plus durable dispatch record → worker extraction through 9router → pending proposal → authenticated decision → atomic financial commit. File writes, SQL and event publication need explicit recovery because they do not share one transaction. Receipt text and model output remain untrusted; plausible arithmetic or model confidence cannot prove the image was read correctly.

First approval fixes image expiry at approval plus seven days. Deny access at expiry and retry physical deletion without extending the deadline after edits, retries, undo or restore. Keep receipt bytes out of workflow history, diagnostics and long-lived backups, and clean acknowledged browser/temp copies. The acceptable deletion delay and rejected/abandoned-image policy still need decisions. See [receipt recovery](./app-architecture.md#receipt--proposal--approved-ledger-change) and [independent dispatch/retention responsibilities](./codebase-structure.md#jobs-files-and-infrastructure).

## Suggested build order

The notes favour working vertical slices over scaffolding the entire proposed tree. Combine the [architecture phases](./app-architecture.md#phased-implementation-and-readiness-gates) with the [newer structure sequence](./codebase-structure.md#implementation-sequence-and-open-choices):

1. Settle owner provisioning, financial edit/reversal rules, transaction abort semantics and the operating/retention contract; pin compatible tools as the first slice needs them.
2. Build owner-only manual ledger capture with exact money, narrow persistence/work contracts and real PostgreSQL migrations. Verify fee accounting, replay, rollback and RLS.
3. Add the PWA outbox and versioned HTTP replay. Exercise lost responses, auth expiry, storage failures and old commands returning after deployment.
4. Add pending proposals and atomic approval, then private receipt/file recovery, independent dispatch and Connect extraction. Verify duplicate/racing decisions and provider failures.
5. Add budgets, recurrence, goals and grounded reports; prove expiry cleanup, missed-work reconciliation and backup restoration.
6. Validate the Terraform-managed Mac runtime and, once funded, AWS deployment. Record actual installed-iPhone behaviour, host recovery and persistent-data evidence before relying on the app.

## Open decisions and later scope

| Decision | What still needs agreement | Owning reference |
|---|---|---|
| Hosting and cost | Mac-only operation versus Mac development/AWS production, uptime/recovery targets, AWS sizing and monthly ceiling. | [Operating decisions](./tech-stack.md#8-decisions-for-user-review) |
| Retention and recovery | Physical deletion bound, sleeping hosts, rejected/abandoned captures, temporary/backup copies and raw-model/audit retention. | [App open decisions](./app-architecture.md#open-decisions) |
| Identity and inference | Owner setup/recovery, real private HTTPS OAuth callback, allowed provider data use, permitted models and measured receipt accuracy. | [Identity and AI boundaries](./tech-stack.md#3-data-correctness-and-authorization) |
| Financial and transaction details | Opening balances, edit/reversal representation, IDR precision/rounding, lock/isolation and rollback contracts, supported offline versions. | [Structure open choices](./codebase-structure.md#implementation-sequence-and-open-choices) |
| Growth | Independent owners versus shared households, admission, workload, network access and per-user limits. Owner columns alone do not establish tenant isolation. | [Small multi-user path](./tech-stack.md#scaling-path-a-small-multi-user-application) |

Later candidates include multi-currency, split transactions, statement imports, read-oriented finance chat, shared budgets and a native client. Automatic approval requires explicit opt-in, representative evaluation and bounded policy; the earlier “about 50 approvals” and confidence threshold are unvalidated heuristics. A separate API service, distributed storage, full offline database replication and extra agent frameworks should follow concrete requirements. See [later product ideas](./agentic-financial-tracker.md#22-later), [deferred tools](./tech-stack.md#alternatives-and-deferred-tools) and [agent authority limits](./app-architecture.md#agent-authority-and-model-boundary).
