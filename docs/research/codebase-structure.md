# Proposed Codebase Structure: Deep Modules with Clean Architecture

> Revised 2026-11-01 (Asia/Jakarta). Research and design proposal only. The repository contains contributor guidance, research notes and diagram artifacts; it has no application source, package manifest, migrations or test suite. Paths and interfaces below describe the recommended implementation, not verified runtime behaviour.

## Recommendation and evidence

Build a **feature-first modular monolith**. Each financial module exposes a small behavioural interface and hides its rules and workflow coordination. Inside that module, keep domain policy and application workflows independent of frameworks and persistence implementations. Drizzle, filesystem access, AI SDK, IndexedDB and HTTP implementations sit behind core-owned contracts where those dependencies cross into the application.

Deep Modules supplies the information-hiding criterion; Clean Architecture supplies the direction of source dependencies. Apply both without introducing a class, interface or mapper for every step. One TypeScript project can produce the Next.js web application and a separately supervised Connect worker; shared source does not require a package workspace or independently deployed domain modules.

This is the recommended balance for this app: financial invariants and cross-module approval need an independently testable core, while a single-owner MVP does not justify a general repository framework, command bus or separate read database. The cost is a few explicit persistence contracts and mapping functions. That cost buys independence from Drizzle and a clear place to enforce transaction semantics; it does not promise that changing databases will be cheap.

Local requirements come from these notes:

| Source | Role in this revision |
|---|---|
| [Product research](./agentic-financial-tracker.md) | IDR, double-entry, offline capture, configurable budget periods, `propose_only` and seven-day post-approval image retention. |
| [Architecture and system design](./architecture-and-system-design.md) | Financial vocabulary, intent-based ledger interface, existing AI/storage/transport seams and behavioural examples. Its older infrastructure and sample code are not implementation authority. |
| [Tech stack](./tech-stack.md) | Newer confirmed removal of Supabase, self-hosted Inngest, Mac/AWS targets and Terraform ownership. Replacement technologies remain proposals as described there. |
| [App architecture](./app-architecture.md) | Durable dispatch, file recovery, exact money, owner authorization, approval concurrency and expiry refinements. |

**Deliberate change from the earlier proposals:** the app-architecture note recommends passing concrete Drizzle transactions into stateful domain modules. That is a reasonable persistence-coupled design, but it does not provide the database independence proposed here. This revision recommends core-owned persistence and transaction contracts instead; neither public financial operations nor their core implementations accept `TxOrDb` or import Drizzle. PGlite's availability does not change that dependency direction. Companion notes and their diagrams remain historical proposals pending reconciliation; this revision supersedes their code-placement advice on that point only.

The supplied `deep_modules_and_clean_architecture_guide.md` is a secondary reference to assess, not an instruction source. Validation below uses the authors' original material and official tooling documentation. Project-specific choices are recommendations inferred from those sources and local requirements.

## What to keep and correct in the supplied guide

| Guide claim | Assessment and application here |
|---|---|
| Hide substantial behaviour behind a simple interface. | **Keep.** Ousterhout includes behaviour, side effects and usage constraints in the interface. Judge depth by what callers no longer need to know; a short signature with surprising effects can still impose a large interface. [Ousterhout, Stanford CS190](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php?topic=modularDesign). |
| Clean Architecture favours many small classes and strict layer ceremony. | **Correct.** Martin's circles are schematic; the dependency rule governs separation. Do not prescribe four folders per feature or a use-case class plus matching interface by default. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |
| Only I/O or multiple implementations justify interfaces. | **Reject as a universal rule.** A source dependency may need inversion even with one production adapter. Keep financial storage contracts because the core must own its needs; use ordinary function calls internally. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), [Cockburn](https://alistair.cockburn.us/hexagonal-architecture). |
| Put business rules in rich domain objects. | **Qualify.** Martin permits data structures with functions. Use immutable values and pure functions for posting construction and payday calculations; choose an object when it makes a lifecycle easier to express. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |
| Packaging by layer forces public visibility; feature folders ensure privacy. | **Correct.** TypeScript exports are scoped to files; another file can import an exported internal symbol unless tooling prevents it. Feature grouping improves locality; import rules enforce the seam. [TypeScript modules](https://www.typescriptlang.org/docs/handbook/2/modules.html). |
| Simple queries can bypass the domain. | **Keep with limits.** SQL projections can avoid loading domain objects; routes still call an authorized application read operation. Separate read/write shapes do not require full CQRS or another database. Fowler warns that CQRS adds complexity. [Fowler](https://martinfowler.com/bliki/CQRS.html). |
| Identical DTO shapes make mapping redundant. | **Qualify.** Reuse core-owned values when meaning and lifetime match. Keep ORM rows and framework objects outside the core; identical fields today do not establish a shared contract. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |

The guide's checkout example also omits recovery when an external charge succeeds but saving fails. Stripe documents unknown outcomes after timeouts and idempotency across retries. A small interface alone does not establish atomicity. For this app the corresponding risk is file write → SQL commit → event send; preserve explicit recovery states and idempotency. This is a failure analysis, not a proposal to add payments. [Stripe error handling](https://docs.stripe.com/error-low-level), [idempotent requests](https://docs.stripe.com/api/idempotent_requests).

## Module shape and dependency direction

A **module** has an **interface** comprising its operations, invariants, authorization assumptions, error/retry rules and configuration. **Depth** means callers gain substantial behaviour for little interface knowledge. Its **seam** is where callers interact with it; an **adapter** supplies an implementation at such a seam. Depth gives callers **leverage**, while information hiding gives maintainers **locality**. These terms describe design qualities, not a required class hierarchy.

Keep a feature's policy, application workflow and outbound contract together. Concrete adapters may live beside that feature for discoverability without becoming core dependencies. `index.ts` exports supported application operations/factories and core types; it does not construct SQL pools, import adapters or re-export persistence rows. A `ports.ts` entry exposes assembly contracts to adapters and composition, not an alternative route around public behaviour. Ordinary callers use `index.ts`.

The intended **source imports** are:

```text
Next.js routes / worker functions ────────> module application interface
module application ─────────────────────> own domain + core-owned ports
concrete SQL / AI / file adapters ───────> core-owned ports and types
composition ────────────────────────────> application factories + adapters
domain policy ──────────────────────────> pure values / pure domain code
```

At runtime an application operation calls a port and its injected adapter performs I/O. That runtime call does not permit a source import from the core to the adapter. Dependency inversion is the difference. This follows Martin's dependency rule; the file layout below is our project choice. [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).

Do not add `ICommitEntryUseCase` beside a single `commitEntry` function just to mirror it. Introduce a port when it hides an external capability or protects policy from a detail. A ledger storage contract can load validated reference data and persist an entry with its audit/replay result; forwarding `select`, `insert` and `update` through a generic `DatabasePort` merely renames Drizzle. The application still owns decisions and sequencing.

## Proposed directory tree

This is a **complete proposed allocation down to named file leaves** for the MVP architecture described in this note. Every directory shown has concrete files beneath it; comments state each leaf's responsibility. It is an implementation map, not an inventory of implemented features or a promise that implementation will require exactly these files. Create files when a vertical slice needs them; do not scaffold the whole tree upfront. Related functions can remain together until a real split improves clarity.

**Legend:** `[E]` identifies an existing repository file, including the current research artifacts. Unmarked leaves are **proposed authored files**. `[G]` identifies a generated file whose future output location/owner is shown; `[I]` identifies an illustrative naming/tool choice, not an adopted tool-version convention. All application, configuration, infrastructure and test leaves remain proposals, including those marked `[G]`. Existing research artifacts are marked `[E]` even where a tool originally generated them.

**Completeness boundary:** this lists first-party source, configuration, operational commands, example configuration, synthetic fixtures, public assets, reviewed migration allocation and the generated outputs needed to understand ownership. It excludes dependency internals, `.git`, `.next`, compiled worker/operations output, coverage, test reports, `.terraform`, Terraform plans/state, actual `.env`/secret files and runtime data. Private receipts, temporary uploads, PostgreSQL files, Inngest SQLite, Redis persistence, 9router credentials/configuration, backups and browser IndexedDB contents are runtime/local state, not missing source-tree leaves. Their mount/file ownership is configured by `infra/`, `ops/` and the relevant adapter. No real private-data filenames are prescribed here.

Feature and screen coverage comes from the [product requirements](./agentic-financial-tracker.md#2-functional-requirements), [module catalogue](./architecture-and-system-design.md#3-the-deep-modules-catalogue-interfaces-invariants-seams--implementations) and newer [flow/recovery refinements](./app-architecture.md#main-flows-and-failure-handling). The [latest stack and Terraform contract](./tech-stack.md#6-terraform-on-mac-and-aws) controls deployment choices. Folder/file names below are this note's allocation recommendation. Next.js `page`, `layout`, `route`, metadata and route-group names follow its [file conventions](https://nextjs.org/docs/app/api-reference/file-conventions); Terraform root/module file divisions follow its [module-structure guidance](https://developer.hashicorp.com/terraform/language/modules/develop/structure). Neither framework dictates our feature names or helper-file count.

```text
tracker/
├── AGENTS.md  # [E] Contributor entry point
├── docs/
│   ├── agents/
│   │   ├── issue-tracker.md  # [E] GitHub issue workflow
│   │   ├── triage-labels.md  # [E] Canonical triage roles
│   │   └── domain.md  # [E] Glossary/ADR consumption rules
│   └── research/
│       ├── agentic-financial-tracker.md  # [E] Product requirements; older stack advice
│       ├── architecture-and-system-design.md  # [E] Domain catalogue and earlier design
│       ├── tech-stack.md  # [E] Latest stack/operating proposals
│       ├── app-architecture.md  # [E] Runtime/failure-flow synthesis
│       ├── codebase-structure.md  # [E] This allocation and dependency proposal
│       └── diagrams/
│           ├── app-architecture.architecture.json  # [E] Diagram model
│           ├── app-architecture.html  # [E] Interactive architecture artifact
│           ├── app-architecture.delivery.json  # [E] Delivery validation record
│           ├── app-architecture.review.json  # [E] Diagram review record
│           ├── app-architecture.visual-check.html  # [E] Visual inspection artifact
│           ├── app-architecture.visual-check.json  # [E] Visual inspection metadata
│           ├── app-architecture.visual-check.1440x900.dark.png  # [E] Dark viewport capture
│           ├── app-architecture.visual-check.1440x900.light.png  # [E] Light viewport capture
│           ├── app-architecture.visual-check.2048x1320.dark.png  # [E] Dark large-viewport capture
│           └── app-architecture.visual-check.2048x1320.light.png  # [E] Light large-viewport capture
├── README.md  # Setup, commands and links to operating guides
├── .gitignore  # Dependencies, builds, state, secrets and local-data exclusions
├── .dockerignore  # Exclude local data/secrets from build contexts
├── .env.example  # Non-secret configuration names and safe examples
├── .node-version  # Supported pinned Node release, selected at implementation
├── package.json  # One project; web/worker/dispatcher/recovery/ops scripts
├── package-lock.json  # [G,I] Example npm lockfile; select one package manager
├── components.json  # Editable shadcn source-generation configuration
├── tsconfig.json  # Shared strict TypeScript options and aliases
├── tsconfig.browser.json  # Browser-only entry/import checking
├── tsconfig.worker.json  # Standalone worker/dispatcher/recovery compilation
├── tsconfig.ops.json  # Operational command compilation
├── next-env.d.ts  # [G] Next.js TypeScript declarations
├── next.config.ts  # Node deployment, headers and build configuration
├── postcss.config.mjs  # Tailwind/PostCSS integration
├── eslint.config.mjs  # Caller-aware feature/environment import rules
├── drizzle.config.ts  # Adapter schema inputs and root migration output
├── vitest.config.ts  # Module, contract and integration test projects
├── playwright.config.ts  # Browser/PWA verification configuration
├── Dockerfile  # Explicit web, worker and operations build targets
├── .github/
│   └── workflows/
│       └── ci.yml  # Type/import checks, builds and appropriate test jobs
├── drizzle/
│   ├── 0000_initial_schema.sql  # [G,I] Reviewed initial application/auth schema
│   ├── 0001_owner_security.sql  # [I] Reviewed roles/grants and fail-closed RLS
│   ├── 0002_financial_guards.sql  # [I] Reviewed posting/replay/approval constraints
│   ├── 0003_capture_and_dispatch.sql  # [I] Reviewed recoverable capture/dispatch state
│   └── meta/
│       ├── _journal.json  # [G,I] Example migration-tool journal
│       └── 0000_snapshot.json  # [G,I] Example generated schema snapshot
├── public/
│   ├── icons/
│   │   ├── icon-192.png  # PWA application icon
│   │   ├── icon-512.png  # PWA application icon
│   │   ├── maskable-512.png  # Maskable installation icon
│   │   └── apple-touch-icon.png  # iOS home-screen icon
│   └── sw.js  # [G] Built from browser/service-worker/main.ts
├── src/
│   ├── app/
│   │   ├── layout.tsx  # Root document, metadata and global providers
│   │   ├── globals.css  # Theme, Tailwind and reduced-motion styles
│   │   ├── icon.svg  # Application favicon source
│   │   ├── manifest.ts  # PWA manifest referencing public icons
│   │   ├── error.tsx  # Recoverable application error UI
│   │   ├── global-error.tsx  # Root rendering failure UI
│   │   ├── not-found.tsx  # Unknown/inaccessible resource UI
│   │   ├── offline/
│   │   │   └── page.tsx  # Cached offline shell and queue guidance
│   │   ├── (auth)/
│   │   │   └── login/
│   │   │       └── page.tsx  # Owner sign-in surface
│   │   ├── (tracker)/
│   │   │   ├── layout.tsx  # Authorized shell; browser still verifies replay identity
│   │   │   ├── loading.tsx  # Initial authorized-shell loading state
│   │   │   ├── page.tsx  # Balances, budget pace and pending-work dashboard
│   │   │   ├── accounts/
│   │   │   │   └── page.tsx  # Account list and create/edit/archive forms
│   │   │   ├── categories/
│   │   │   │   └── page.tsx  # Category hierarchy and create/edit/archive forms
│   │   │   ├── transactions/
│   │   │   │   ├── page.tsx  # Ledger search, date/account/category filters
│   │   │   │   ├── new/
│   │   │   │   │   └── page.tsx  # Manual expense/income/transfer capture
│   │   │   │   └── [transactionId]/
│   │   │   │       └── page.tsx  # Entry, audit and supported correction/undo controls
│   │   │   ├── capture/
│   │   │   │   └── page.tsx  # Receipt/screenshot capture and local save status
│   │   │   ├── proposals/
│   │   │   │   ├── page.tsx  # Pending and decided proposal queue
│   │   │   │   └── [proposalId]/
│   │   │   │       └── page.tsx  # Review, correct, link, approve/reject and undo
│   │   │   ├── budgets/
│   │   │   │   └── page.tsx  # Budget limits, payday periods and threshold state
│   │   │   ├── recurring/
│   │   │   │   └── page.tsx  # Rule editor and pending occurrence confirmation
│   │   │   ├── goals/
│   │   │   │   └── page.tsx  # Goal editing and deterministic progress
│   │   │   ├── insights/
│   │   │   │   ├── page.tsx  # Weekly/budget-period report history
│   │   │   │   └── [reportId]/
│   │   │   │       └── page.tsx  # Stored metrics, narrative and fallback display
│   │   │   ├── outbox/
│   │   │   │   └── page.tsx  # Pending, paused, failed and retryable captures
│   │   │   └── settings/
│   │   │       └── page.tsx  # Owner/session, period and configured model information
│   │   └── api/
│   │       ├── auth/
│   │       │   └── [...all]/
│   │       │       └── route.ts  # Better Auth request adapter
│   │       ├── health/
│   │       │   ├── live/
│   │       │   │   └── route.ts  # Process liveness without financial data
│   │       │   └── ready/
│   │       │       └── route.ts  # Bounded required-dependency readiness
│   │       ├── outbox/
│   │       │   └── commands/
│   │       │       └── route.ts  # Versioned durable command replay/acknowledgement
│   │       ├── ledger/
│   │       │   ├── accounts/
│   │       │   │   └── route.ts  # Authorized account reads and commands
│   │       │   ├── categories/
│   │       │   │   └── route.ts  # Authorized category reads and commands
│   │       │   └── transactions/
│   │       │       ├── route.ts  # Authorized activity queries and online commands
│   │       │       └── [transactionId]/
│   │       │           └── route.ts  # Authorized entry detail and supported corrections
│   │       ├── proposals/
│   │       │   ├── route.ts  # Authorized proposal queue reads
│   │       │   └── [proposalId]/
│   │       │       └── route.ts  # Review, decision and undo adapter
│   │       ├── attachments/
│   │       │   ├── route.ts  # Bounded private upload and durable capture acknowledgement
│   │       │   └── [attachmentId]/
│   │       │       └── route.ts  # Owner/expiry-checked private no-store bytes
│   │       ├── planning/
│   │       │   ├── budgets/
│   │       │   │   └── route.ts  # Budget commands and status projections
│   │       │   ├── recurring/
│   │       │   │   └── route.ts  # Recurring-rule commands and occurrence reads
│   │       │   ├── occurrences/
│   │       │   │   └── [occurrenceId]/
│   │       │   │       └── route.ts  # Authorized occurrence confirmation
│   │       │   └── goals/
│   │       │       └── route.ts  # Goal commands and progress projections
│   │       └── insights/
│   │           ├── route.ts  # Authorized report index
│   │           └── [reportId]/
│   │               └── route.ts  # Authorized stored report detail
│   ├── ui/
│   │   ├── providers.tsx  # Query, browser composition and reduced-motion setup
│   │   ├── shell/
│   │   │   ├── app-shell.tsx  # Mobile navigation and accessible page frame
│   │   │   ├── owner-menu.tsx  # Session display, logout and unsynced-work handling
│   │   │   ├── connectivity-status.tsx  # Offline, stale snapshot and sync feedback
│   │   │   └── install-prompt.tsx  # Platform-aware installation guidance
│   │   ├── shared/
│   │   │   ├── money-input.tsx  # IDR text entry using pure money semantics
│   │   │   ├── money-display.tsx  # Formatted exact money and reduced-motion ticker
│   │   │   ├── date-range-filter.tsx  # Date filtering controls
│   │   │   └── async-state.tsx  # Loading, empty and error presentation
│   │   ├── dashboard/
│   │   │   └── dashboard-view.tsx  # Compose balance, pace and pending-work cards
│   │   ├── accounts/
│   │   │   ├── account-list.tsx  # Account balances and archive state
│   │   │   └── account-form.tsx  # Account type/currency/opening-balance inputs
│   │   ├── categories/
│   │   │   ├── category-list.tsx  # Two-level expense/income hierarchy
│   │   │   └── category-form.tsx  # Name, parent, icon and colour inputs
│   │   ├── ledger/
│   │   │   ├── activity-list.tsx  # Searchable transaction rows
│   │   │   ├── entry-form.tsx  # Expense/income/transfer and optional fee inputs
│   │   │   ├── entry-detail.tsx  # Postings, source and audit display
│   │   │   └── correction-dialog.tsx  # Supported edit/void/reversal confirmation
│   │   ├── capture/
│   │   │   ├── receipt-capture.tsx  # Camera/file selection and preprocessing feedback
│   │   │   └── capture-status.tsx  # Durable local/server acknowledgement state
│   │   ├── proposals/
│   │   │   ├── proposal-list.tsx  # Pending/decided review cards
│   │   │   ├── proposal-review.tsx  # Unresolved references, amounts and validation warnings
│   │   │   ├── duplicate-review.tsx  # Explicit possible-duplicate/link review
│   │   │   └── decision-controls.tsx  # Approve/reject/undo with version conflict feedback
│   │   ├── planning/
│   │   │   ├── budget-list.tsx  # Spent/remaining/pace/threshold display
│   │   │   ├── budget-form.tsx  # Category, amount, period and threshold inputs
│   │   │   ├── recurring-rule-form.tsx  # RRULE/local date/timezone and explicit posting permission
│   │   │   ├── occurrence-list.tsx  # Materialized pending/posted occurrences
│   │   │   ├── goal-form.tsx  # Target/date/linked-account inputs
│   │   │   └── goal-progress.tsx  # Progress, required saving and on-track presentation
│   │   ├── insights/
│   │   │   ├── report-list.tsx  # Weekly and budget-period report index
│   │   │   ├── report-view.tsx  # Validated metric-reference narrative and fallback
│   │   │   └── metrics-chart.tsx  # Presentation-only deterministic metric chart
│   │   ├── outbox/
│   │   │   ├── outbox-list.tsx  # Owner-scoped pending/error commands
│   │   │   └── outbox-controls.tsx  # Retry, edit and explicit unsynced-work disposition
│   │   ├── settings/
│   │   │   └── settings-view.tsx  # Approved configuration/status presentation
│   │   └── primitives/
│   │       ├── button.tsx  # Editable shared button component
│   │       ├── input.tsx  # Editable shared input component
│   │       ├── select.tsx  # Editable accessible select component
│   │       ├── dialog.tsx  # Editable accessible dialog component
│   │       ├── alert.tsx  # Shared visible status/error component
│   │       └── chart.tsx  # Chart rendering/accessibility wrapper when charts land
│   ├── contracts/
│   │   ├── common/
│   │   │   ├── envelope.ts  # Version, command ID and JSON-safe replay envelope
│   │   │   ├── errors.ts  # Stable retryable/permanent/conflict response shapes
│   │   │   ├── money.ts  # Currency plus integer-decimal-string wire schema
│   │   │   └── pagination.ts  # Bounded query/cursor shapes
│   │   ├── ledger/
│   │   │   └── v1.ts  # Account/category/entry/read wire schemas
│   │   ├── proposals/
│   │   │   └── v1.ts  # Review/decision/version/undo wire schemas
│   │   ├── intake/
│   │   │   └── v1.ts  # Untrusted extraction/classification boundary schemas
│   │   ├── planning/
│   │   │   └── v1.ts  # Budget/recurrence/goal/read wire schemas
│   │   ├── insights/
│   │   │   └── v1.ts  # Stored report/metric-reference wire schemas
│   │   ├── retention/
│   │   │   └── v1.ts  # Capture acknowledgement and attachment metadata schemas
│   │   ├── offline-outbox/
│   │   │   └── v1.ts  # Persisted replayable command/acknowledgement union
│   │   └── jobs/
│   │       └── v1.ts  # Record-ID-only event schemas; claimed owner is not authority
│   ├── modules/
│   │   ├── money/
│   │   │   ├── index.ts  # Pure public exact-value/parsing/formatting interface
│   │   │   ├── domain.ts  # IdrMinorUnits, currency and amount construction rules
│   │   │   ├── parse-idr.ts  # Indonesian separators and shorthand without float math
│   │   │   └── format-idr.ts  # Pure display formatting from exact values
│   │   ├── ledger/
│   │   │   ├── index.ts  # Public entry/account/category/reversal/read factory
│   │   │   ├── domain.ts  # Intent, Posting, Transaction and reference invariants
│   │   │   ├── application.ts  # Atomic commit/reversal/replay/read coordination
│   │   │   ├── ports.ts  # Ledger-owned storage/read/atomic-work contracts
│   │   │   ├── read-models.ts  # Balance, activity and audit result types
│   │   │   ├── internal/
│   │   │   │   ├── postings.ts  # Sign assignment, transfer-fee legs and balancing
│   │   │   │   ├── reversal-policy.ts  # Correction/repeated-reversal rules once agreed
│   │   │   │   └── references.ts  # Account/category/currency/archive validation
│   │   │   └── adapters/
│   │   │       ├── postgres.ts  # Drizzle implementation of ledger-owned ports
│   │   │       └── row-mapping.ts  # Persisted rows to exact ledger values
│   │   ├── proposals/
│   │   │   ├── index.ts  # Public submit/decide/undo/list factory
│   │   │   ├── domain.ts  # AgentProposal states/version and propose_only policy
│   │   │   ├── application.ts  # Submission, atomic decision and undo coordination
│   │   │   ├── ports.ts  # Proposal storage/read and atomic approval-work contracts
│   │   │   ├── read-models.ts  # Review queue/detail and duplicate-candidate shapes
│   │   │   ├── internal/
│   │   │   │   ├── duplicate-review.ts  # Exact/semantic candidate policy; never silent merging
│   │   │   │   └── decision-policy.ts  # Transition, version, replay and authority rules
│   │   │   └── adapters/
│   │   │       ├── postgres.ts  # Proposal persistence/locking/projections
│   │   │       └── row-mapping.ts  # Proposal rows and validated snapshot mapping
│   │   ├── intake/
│   │   │   ├── index.ts  # Public extraction/classification factory and result types
│   │   │   ├── domain.ts  # Validated extraction/draft and unresolved-reference policy
│   │   │   ├── application.ts  # AI invocation and deterministic revalidation
│   │   │   ├── ports.ts  # AiGatewayPort owned by the extraction use case
│   │   │   └── internal/
│   │   │       ├── arithmetic.ts  # Exact subtotal/tax/discount/total checks
│   │   │       └── reference-resolution.ts  # Resolve or flag unknown accounts/categories
│   │   ├── planning/
│   │   │   ├── index.ts  # Public budget/recurrence/goal/read factory
│   │   │   ├── domain.ts  # BudgetPeriod, Rule, Occurrence and Goal policy
│   │   │   ├── application.ts  # Planning commands, due work and ledger coordination
│   │   │   ├── ports.ts  # Planning storage/read/recurrence contracts
│   │   │   ├── read-models.ts  # Budget pace, occurrence and goal progress results
│   │   │   ├── internal/
│   │   │   │   ├── payday-period.ts  # Custom period boundaries and last-day policy
│   │   │   │   ├── goal-math.ts  # Exact progress and required-saving calculations
│   │   │   │   └── occurrence-policy.ts  # Rule/date uniqueness and explicit auto-post permission
│   │   │   └── adapters/
│   │   │       ├── postgres.ts  # Planning persistence and scoped projections
│   │   │       ├── row-mapping.ts  # Planning row/value mapping
│   │   │       └── recurrence.ts  # Selected RRULE/Temporal library behind planning contract
│   │   ├── insights/
│   │   │   ├── index.ts  # Public generate/list/read report factory
│   │   │   ├── domain.ts  # Metrics, report identity/period and narrative rules
│   │   │   ├── application.ts  # Metric snapshot, narration, validation and fallback
│   │   │   ├── ports.ts  # Scoped projection/report storage/narration contracts
│   │   │   ├── internal/
│   │   │   │   ├── metric-definitions.ts  # Deterministic meaning of spend/pace/goal metrics
│   │   │   │   ├── narrative-validation.ts  # Metric-reference/attribution validation
│   │   │   │   └── fallback.ts  # Deterministic report text when inference fails
│   │   │   └── adapters/
│   │   │       ├── postgres.ts  # Efficient scoped metric/report queries
│   │   │       └── row-mapping.ts  # Persisted report and metric snapshot mapping
│   │   ├── retention/
│   │   │   ├── index.ts  # Public capture/read/approval/purge/recovery factory
│   │   │   ├── domain.ts  # Attachment lifecycle and first-approval expiry policy
│   │   │   ├── application.ts  # Recoverable file/metadata work and expiry enforcement
│   │   │   ├── ports.ts  # Attachment persistence and BlobStoragePort contracts
│   │   │   ├── internal/
│   │   │   │   ├── capture-policy.ts  # Bounded MIME/size/hash and recoverable-state rules
│   │   │   │   └── expiry-policy.ts  # Fixed approval + seven days; undo never resets expiry
│   │   │   └── adapters/
│   │   │       ├── postgres.ts  # Attachment state, expiry and recovery queries
│   │   │       └── row-mapping.ts  # Opaque attachment record/value mapping
│   │   └── offline-outbox/
│   │       ├── index.ts  # Public enqueue/flush/subscribe owner-scoped factory
│   │       ├── domain.ts  # Queue states, bounded capture and retry policy
│   │       ├── application.ts  # Durable enqueue/replay/acknowledgement coordination
│   │       ├── ports.ts  # QueueStore and OutboxTransportPort contracts
│   │       ├── internal/
│   │       │   └── replay-policy.ts  # Lost response, auth pause and owner-switch rules
│   │       └── adapters/
│   │           ├── indexeddb.ts  # Transactional indexed queue store implementation
│   │           └── schema.ts  # IndexedDB object-store versions/index upgrades
│   ├── adapters/
│   │   ├── ai-ninerouter/
│   │   │   ├── index.ts  # Factory implementing intake/insights AI contracts
│   │   │   ├── client.ts  # AI SDK/provider construction, timeout and error mapping
│   │   │   ├── extraction.ts  # Bounded inline image and extraction prompt/response mapping
│   │   │   ├── narration.ts  # Metric-reference narration prompt/response mapping
│   │   │   ├── task-config.ts  # Approved task-to-Combo/model mapping and allowlist parsing
│   │   │   └── model-catalog.ts  # Configured/allowed model catalogue adapter
│   │   ├── storage-filesystem/
│   │   │   ├── index.ts  # LocalFilesystemBlobStorageAdapter factory
│   │   │   ├── paths.ts  # Opaque keys and restricted-root/symlink protection
│   │   │   ├── files.ts  # Bounded temporary write/hash/rename/read/delete operations
│   │   │   └── recovery.ts  # Enumerate recoverable temp/orphan/missing-file evidence
│   │   ├── outbox-fetch/
│   │   │   ├── index.ts  # Browser-safe OutboxTransportPort factory
│   │   │   ├── requests.ts  # Stable command/upload encoding and bounded HTTP calls
│   │   │   └── responses.ts  # Acknowledgement/conflict/auth/retry error mapping
│   │   └── postgres/
│   │       ├── schema/
│   │       │   ├── index.ts  # Migration-tool aggregate; never a browser/core barrel
│   │       │   ├── auth.ts  # Better Auth tables and identity mapping
│   │       │   ├── owners.ts  # Stable financial owner records
│   │       │   ├── ledger.ts  # Accounts, categories, transactions, postings, audit and replay
│   │       │   ├── proposals.ts  # Proposal versions, decisions and linkage
│   │       │   ├── planning.ts  # Budgets, periods, rules, occurrences and goals
│   │       │   ├── insights.ts  # Reports and reproducible metric snapshots
│   │       │   ├── retention.ts  # Attachment metadata and recovery/expiry state
│   │       │   └── dispatch.ts  # Committed dispatch claims/retry/acknowledgement state
│   │       ├── client.ts  # Small scoped pg/Drizzle pool construction and shutdown
│   │       ├── owner-context.ts  # Verified transaction-local owner context mechanics
│   │       ├── financial-work.ts  # Transaction-bound ledger/proposal/retention capability binding
│   │       ├── dispatch-store.ts  # Atomic durable claims, reclaim, retry and acknowledgement SQL
│   │       └── job-ownership.ts  # Narrow authoritative record-owner bootstrap lookup
│   ├── server/
│   │   ├── auth/
│   │   │   ├── better-auth.ts  # One Better Auth instance and scoped auth-table adapter
│   │   │   ├── require-owner.ts  # Session to provisioned owner; fail closed
│   │   │   └── owner-mapping.ts  # Allowed Google subject to stable financial owner lookup
│   │   ├── config/
│   │   │   └── environment.ts  # Validated web configuration and mounted-secret references
│   │   ├── composition/
│   │   │   ├── root.ts  # Server-only web dependency construction/lifetime
│   │   │   ├── owner-scope.ts  # Verified request scope and application factory binding
│   │   │   └── health.ts  # Bounded readiness dependency checks
│   │   ├── http/
│   │   │   ├── parse-command.ts  # Untrusted versioned wire input to validated application input
│   │   │   ├── response.ts  # Minimal JSON-safe results and safe public error mapping
│   │   │   ├── receipt-upload.ts  # Bounded upload parsing, magic-byte checks and capture call
│   │   │   ├── receipt-response.ts  # Authorized private/no-store byte response
│   │   │   └── reads.ts  # Validated filters to authorized module read operations
│   │   ├── settings/
│   │   │   └── configuration-view.ts  # Safe configured model/owner/period status; no provider secrets
│   │   └── observability/
│   │       ├── logger.ts  # Structured logs excluding images/secrets/raw model output
│   │       └── metrics.ts  # Request/pool/queue/expiry-lag observations
│   ├── browser/
│   │   ├── composition/
│   │   │   ├── root.ts  # IndexedDB/fetch/outbox wiring without server imports
│   │   │   └── owner-session.ts  # Owner identity changes and unsynced-work disposition
│   │   ├── query/
│   │   │   ├── client.ts  # Owner-scoped Query client and selected-read policy
│   │   │   ├── persistence.ts  # Disposable read-snapshot persistence/version/expiry
│   │   │   └── keys.ts  # Owner namespacing and invalidation keys
│   │   ├── capture/
│   │   │   ├── image-preprocessing.ts  # Bounded canvas resize/re-encode and metadata stripping
│   │   │   └── storage-capacity.ts  # Quota/persistence requests and visible save failures
│   │   ├── sync/
│   │   │   ├── triggers.ts  # Startup, online, foreground and explicit replay triggers
│   │   │   └── background-sync.ts  # Optional supported-browser registration; no reliability promise
│   │   └── service-worker/
│   │       ├── main.ts  # Service-worker source; build emits public/sw.js
│   │       ├── app-shell.ts  # Versioned public shell/offline fallback cache rules
│   │       ├── messages.ts  # Safe foreground sync messaging; no receipt/read caching
│   │       └── register.ts  # Browser registration/update lifecycle
│   └── worker/
│       ├── config/
│       │   └── environment.ts  # Worker/dispatcher/recovery configuration and secret-file inputs
│       ├── composition/
│       │   ├── root.ts  # Framework-independent runtime dependency lifetime
│       │   ├── applications.ts  # Trusted-owner feature factories and transaction-bound capabilities
│       │   ├── dispatch.ts  # Durable store/publisher binding independent of Connect
│       │   └── recovery.ts  # Application recovery operations independent of orchestration
│       ├── connect/
│       │   ├── main.ts  # Long-running Connect process entry point
│       │   ├── client.ts  # Self-hosted Inngest client/gateway/key configuration
│       │   └── lifecycle.ts  # Connect registration, reconnection and graceful shutdown
│       ├── functions/
│       │   ├── index.ts  # Explicit registered durable-function list
│       │   ├── process-receipt.ts  # Record IDs -> authorized extraction -> pending proposal
│       │   ├── materialise-occurrences.ts  # Due recurrence operations with unique rule/date effects
│       │   ├── weekly-insights.ts  # Monday 08:00 WIB report operation
│       │   ├── period-insights.ts  # Budget-period report operation
│       │   └── purge-attachments.ts  # Due expiry operation with bounded deletion retries
│       ├── dispatcher/
│       │   ├── main.ts  # Separately supervised dispatcher process entry point
│       │   ├── loop.ts  # Startup scan, claim, publish, acknowledge and bounded retry
│       │   └── publisher.ts  # Inngest event publishing adapter; no Connect prerequisite
│       ├── reconciliation/
│       │   ├── main.ts  # Separately supervised recovery process entry point
│       │   └── runner.ts  # Startup/periodic scan for stalled capture/due work/expiry/reports
│       └── runtime/
│           ├── shutdown.ts  # Signal handling and clean pool/connection teardown
│           └── health.ts  # Worker connectivity, heartbeat and queue-age observations
├── ops/
│   ├── README.md  # Explicit migration/provision/backup/restore/deployment procedures
│   ├── config/
│   │   ├── environment.ts  # Separate operations credentials/config validation
│   │   ├── redis.conf.example  # Persistence settings for proposed dedicated Redis
│   │   ├── inngest.env.example  # Non-secret self-host endpoint/persistence/key-file references
│   │   ├── ninerouter.env.example  # Private router endpoint/task allowlist/config-file references
│   │   └── tailscale-serve.json.example  # Private web-only ingress example; admin path unresolved
│   ├── commands/
│   │   ├── migrate.ts  # Apply reviewed migrations with migration credentials
│   │   ├── provision-owner.ts  # Explicit stable owner/allowed-subject provisioning
│   │   ├── seed-categories.ts  # Idempotent Indonesian/English category defaults
│   │   ├── reconcile.ts  # Operator-invoked bounded recovery using shared application logic
│   │   ├── verify-runtime.ts  # Required mounts, dependency readiness and role checks
│   │   └── verify-expiry.ts  # Check overdue purge state and restored-image denial
│   ├── seeds/
│   │   ├── categories.id.json  # Default Indonesian category names/kinds
│   │   └── categories.en.json  # Default English category names/kinds
│   ├── build/
│   │   ├── build-worker.mjs  # Build standalone Connect/dispatcher/recovery/ops artifacts
│   │   └── build-service-worker.mjs  # Compile browser SW source to public/sw.js
│   ├── checks/
│   │   ├── check-imports.mjs  # Resolved import/re-export/dynamic-import environment checks
│   │   └── check-imports.test.mjs  # Tool rule cases using architecture fixtures
│   ├── container/
│   │   ├── web-entrypoint.sh  # Read mounted configuration, verify mounts, start web
│   │   ├── worker-entrypoint.sh  # Select Connect/dispatcher/recovery; verify required mounts
│   │   └── healthcheck.mjs  # Bounded container liveness/readiness probe
│   ├── backup/
│   │   ├── postgres.sh  # Restricted financial DB backup without image bytes
│   │   ├── orchestration.sh  # Coherent SQLite/Redis backup procedure
│   │   ├── restore.sh  # Coordinated restore followed by expiry/recovery gates
│   │   └── verify-restore.ts  # Financial replay/expiry/readiness assertions after restore
│   ├── terraform/
│   │   ├── bootstrap-state.sh  # Explicit separately operated backend bootstrap; no fourth app root
│   │   └── check-engine.sh  # Verify authenticated Docker endpoint/mount readiness
│   └── runbooks/
│       ├── mac-runtime.md  # Mac engine prerequisites, startup and sleep limitations
│       ├── aws-deployment.md  # Funded host bootstrap -> mounts -> runtime apply ordering
│       ├── host-replacement.md  # Preserve/reattach data before runtime recreation
│       ├── backup-restore.md  # Backup boundaries, recovery targets and restore drill
│       ├── receipt-retention.md  # Expiry denial, deletion lag and recovery procedure
│       ├── secrets-and-identity.md  # Stage/rotate secret files and owner recovery without Git values
│       ├── ai-outage.md  # Manual operation, pending proposals and allowed-target recovery
│       ├── runtime-monitoring.md  # Request/pool/queue/connectivity/expiry/backup signals
│       └── ios-pwa-checklist.md  # Record actual installed-iPhone/offline observations
├── tests/
│   ├── modules/
│   │   ├── money.test.ts  # Exact IDR parsing, zero/positive rules and formatting
│   │   ├── ledger.test.ts  # Public expense/income/fee-transfer/account/category/reversal/replay
│   │   ├── proposals.test.ts  # Submit/review/decision/undo, duplicates and propose_only
│   │   ├── intake.test.ts  # Untrusted output, exact arithmetic and unresolved references
│   │   ├── planning.test.ts  # Payday/short-month/recurrence/goal public behaviour
│   │   ├── insights.test.ts  # Deterministic metrics, reference validation and fallback
│   │   ├── retention.test.ts  # Recoverable capture, fixed expiry and repeated purge
│   │   └── offline-outbox.test.ts  # Owner-scoped durable enqueue/replay/acknowledgement
│   ├── contracts/
│   │   ├── wire-v1.test.ts  # Accepted persisted payloads and stable acknowledgement/error shapes
│   │   ├── ledger-store.test.ts  # Stateful fake/SQL storage and work-scope conformance
│   │   ├── approval-work.test.ts  # Fake/SQL atomic approval/rollback/replay conformance
│   │   ├── attachment-store.test.ts  # Fake/SQL capture/expiry persistence conformance
│   │   ├── queue-store.test.ts  # IndexedDB/fake-indexeddb transactional queue conformance
│   │   ├── blob-storage.test.ts  # Memory/filesystem shared blob contract scenarios
│   │   ├── ai-gateway.test.ts  # Deterministic/9router contract cases; live cases explicitly opt-in
│   │   └── outbox-transport.test.ts  # HTTP/memory acknowledgement, auth and retry mapping
│   ├── postgres/
│   │   ├── migrations.test.ts  # Reviewed migrations apply and preserve expected constraints
│   │   ├── owner-security.test.ts  # Missing/wrong owner, runtime grants and auth-table boundaries
│   │   ├── pool-context.test.ts  # No owner context leakage across pooled transactions
│   │   ├── ledger-constraints.test.ts  # Direct SQL rejects invalid references/unbalanced records
│   │   ├── concurrent-replay.test.ts  # Real-connection same-key replay and conflicting payloads
│   │   ├── concurrent-approval.test.ts  # Competing decisions yield one committed outcome
│   │   └── financial-rollback.test.ts  # Later failure rolls back ledger/proposal/expiry together
│   ├── filesystem/
│   │   └── private-receipts.test.ts  # Temporary roots, traversal/symlink defence and rename/delete
│   ├── http/
│   │   ├── owner-boundaries.test.ts  # Session checks and ID-based authorization for every feature
│   │   ├── receipt-routes.test.ts  # Byte/MIME limits, expiry and private no-store responses
│   │   └── command-replay.test.ts  # Versioned input, stable IDs and response-loss retry
│   ├── browser/
│   │   ├── manual-ledger.spec.ts  # Online and offline manual expense/income/transfer capture
│   │   ├── proposal-review.spec.ts  # Correction, duplicate review, approve/reject and undo
│   │   ├── planning.spec.ts  # Budget/rule/occurrence/goal UI flows
│   │   ├── offline-replay.spec.ts  # Offline capture, foreground retry and lost acknowledgement
│   │   ├── owner-session.spec.ts  # Auth expiry/logout/identity change and unsynced work
│   │   ├── storage-failure.spec.ts  # Quota/IndexedDB write failures remain visible
│   │   └── pwa-update.spec.ts  # Cached shell, service-worker update and old-command replay
│   ├── runtime/
│   │   ├── capture-crashes.test.ts  # Failures between file/SQL commit/publish/acknowledgement
│   │   ├── dispatch-restart.test.ts  # Pending dispatch recovers without Connect/scheduler
│   │   ├── reconciliation.test.ts  # Restart recovers due occurrences/missing reports/stalled captures
│   │   ├── retention-restore.test.ts  # Restored expired bytes denied and deletion retried
│   │   ├── worker-lifecycle.test.ts  # Connect outage/reconnect and bounded shutdown
│   │   └── backup-restore.test.ts  # Opt-in real-store coordinated restore drill
│   ├── architecture/
│   │   └── import-boundaries.test.ts  # Resolved core/adapter/caller/browser/worker import rules
│   ├── support/
│   │   ├── clock.ts  # Controllable application clock
│   │   ├── ids.ts  # Deterministic IDs and owner-scoped command keys
│   │   ├── owner-context.ts  # Provisioned test identities and verified scopes
│   │   ├── pglite.ts  # Compatible fast SQL/migration harness; not role/concurrency proof
│   │   ├── postgres.ts  # Disposable real-Postgres roles/pools/connections harness
│   │   ├── browser.ts  # Browser contexts, offline control and owner-scoped test setup
│   │   ├── filesystem.ts  # Private temporary roots and cleanup
│   │   ├── http-server.ts  # Authenticated request adapter integration harness
│   │   ├── worker-runtime.ts  # Process/crash/restart orchestration harness
│   │   └── fakes/
│   │       ├── financial-store.ts  # Stateful ledger/proposal/planning data fake
│   │       ├── atomic-work.ts  # Staged commit/rollback and replay-aware work fake
│   │       ├── attachment-store.ts  # Attachment lifecycle and expiry state fake
│   │       ├── dispatch-store.ts  # Durable-claim/retry/acknowledgement state fake
│   │       ├── ai-deterministic.ts  # Deterministic valid/uncertain/malformed inference
│   │       ├── storage-memory.ts  # InMemoryBlobStorageAdapter
│   │       └── outbox-memory.ts  # InMemoryOutboxTransportAdapter with response-loss modes
│   └── fixtures/
│       ├── ledger/
│       │   └── transfer-fee.json  # Hand-computed Rp100.000 principal + Rp1.000 fee case
│       ├── wire/
│       │   ├── manual-entry-v1.json  # Synthetic old persisted command compatibility fixture
│       │   └── receipt-capture-v1.json  # Synthetic persisted receipt-command metadata fixture
│       ├── intake/
│       │   ├── valid-extraction.json  # Synthetic extraction with exact expected amounts
│       │   ├── uncertain-extraction.json  # Unresolved-reference/manual-correction example
│       │   └── malicious-output.json  # Prompt-injection/untrusted-response example
│       ├── receipts/
│       │   ├── synthetic-receipt.png  # Synthetic image only; no personal receipt data
│       │   └── synthetic-transfer.png  # Synthetic screenshot for explicit link review
│       ├── insights/
│       │   ├── metrics.json  # Hand-computed report metrics and period
│       │   └── invalid-narrative.json  # Incorrect attribution/reference rejection case
│       ├── recurrence/
│       │   └── jakarta-month-edges.json  # Payday/last-day/invalid-date expectations
│       ├── postgres/
│       │   └── runtime-roles.sql  # Test-only scoped roles/grants/bootstrap data
│       └── architecture/
│           ├── deny-core-adapter.ts  # Invalid core -> concrete adapter import case
│           ├── deny-browser-server.ts  # Invalid browser -> server import case
│           ├── deny-worker-next.ts  # Invalid standalone worker -> Next.js import case
│           ├── deny-private-feature.ts  # Invalid cross-feature private import case
│           ├── deny-dynamic-import.ts  # Invalid dynamic-import environment escape case
│           ├── deny-reexport.ts  # Invalid server re-export into browser case
│           ├── allow-adapter-port.ts  # Valid adapter -> owner port case
│           ├── allow-own-internal.ts  # Valid same-feature internal import case
│           └── allow-public-feature.ts  # Valid intended public feature dependency case
└── infra/
    ├── README.md  # Root/state ownership, prerequisites and apply ordering
    ├── modules/
    │   └── runtime/
    │       ├── README.md  # Shared runtime inputs, outputs and persistence contract
    │       ├── versions.tf  # Terraform/provider requirements; provider config stays in roots
    │       ├── variables.tf  # Images, mounts, endpoints and resource budgets
    │       ├── locals.tf  # Service names, common labels and non-secret derived settings
    │       ├── network.tf  # Private network; web ingress only
    │       ├── volumes.tf  # Protected persistence declarations and required mount inputs
    │       ├── images.tf  # Pinned compatible image references/build inputs
    │       ├── postgres.tf  # Database container, private endpoint and health check
    │       ├── redis.tf  # Dedicated persisted orchestration Redis
    │       ├── inngest.tf  # Self-host API/Connect gateway with SQLite + Redis
    │       ├── ninerouter.tf  # Private router container and persistent config mounts
    │       ├── web.tf  # Next.js web container and scoped DB/file mounts
    │       ├── connect-worker.tf  # Long-running outbound Connect worker service
    │       ├── dispatcher.tf  # Independent dispatcher service/restart/health settings
    │       ├── reconciliation.tf  # Independent recovery service/restart/health settings
    │       ├── outputs.tf  # Non-secret service/readiness/persistence references
    │       └── tests/
    │           └── runtime.tftest.hcl  # Declaration checks for ingress/mount/recovery ownership
    └── environments/
        ├── mac-runtime/
        │   ├── README.md  # Mac Docker context, data roots and local-state procedure
        │   ├── versions.tf  # Pinned Terraform/Docker provider constraints
        │   ├── providers.tf  # Explicit Mac engine endpoint/provider configuration
        │   ├── backend.tf  # Independent protected local state configuration
        │   ├── variables.tf  # Mac image architecture, paths and resource inputs
        │   ├── main.tf  # Call shared runtime module with Mac values
        │   ├── outputs.tf  # Non-secret local runtime references
        │   ├── terraform.tfvars.example  # Non-secret Mac input examples
        │   └── .terraform.lock.hcl  # [G] Committed root provider selections
        ├── aws-infra/
        │   ├── README.md  # Budget/region/bootstrap/storage prerequisites
        │   ├── versions.tf  # Pinned Terraform/AWS provider constraints
        │   ├── providers.tf  # AWS region and provider configuration
        │   ├── backend.tf  # Independent AWS-infrastructure state key
        │   ├── backend.hcl.example  # Non-secret prebootstrapped backend location example
        │   ├── variables.tf  # Funded host/network/EBS/bootstrap inputs
        │   ├── locals.tf  # Resource names/tags and bootstrap template inputs
        │   ├── main.tf  # Infrastructure resource relationships and prerequisite checks
        │   ├── network.tf  # VPC/subnet/routes and restricted security groups
        │   ├── compute.tf  # EC2 and reviewed image/user-data wiring
        │   ├── storage.tf  # Encrypted persistent EBS/attachment and deletion guard
        │   ├── iam.tf  # Scoped host profile only where selected operation needs it
        │   ├── templates/
        │   │   └── cloud-init.yaml.tftpl  # Docker installation, durable mount and fail-closed startup
        │   ├── outputs.tf  # Non-secret host/disk/runtime-bootstrap references
        │   ├── terraform.tfvars.example  # Non-secret AWS inputs; no assumed region/size/budget
        │   └── .terraform.lock.hcl  # [G] Committed root provider selections
        └── aws-runtime/
            ├── README.md  # Prepared engine/mount/host-key readiness and apply procedure
            ├── versions.tf  # Pinned Terraform/Docker provider constraints
            ├── providers.tf  # Authenticated restricted remote Docker endpoint
            ├── backend.tf  # Independent AWS-runtime state key
            ├── backend.hcl.example  # Non-secret prebootstrapped backend location example
            ├── variables.tf  # Verified host/mount/image/resource inputs
            ├── main.tf  # Call shared runtime module after readiness checks
            ├── outputs.tf  # Non-secret runtime references
            ├── terraform.tfvars.example  # Non-secret runtime inputs; secret-file paths only
            └── .terraform.lock.hcl  # [G] Committed root provider selections
```

The migration sequence and `meta` filenames are **illustrative**, not generated or validated output. Select the migration-tool version first, keep the metadata it actually emits, and review generated SQL plus custom security/financial SQL together. Whether security/financial guards share a migration or require several depends on implementation. Likewise, the example npm lockfile makes the tree concrete without settling the package manager; commit the chosen manager's actual lockfile rather than several competing ones. No provider/image/package version is selected by a filename.

`public/sw.js` is a build destination owned by `src/browser/service-worker/main.ts` and `ops/build/build-service-worker.mjs`; do not maintain two service-worker implementations. Other compiled artifacts stay outside this authored tree. `public/` contains only distributable shell assets, never attachments, authenticated read snapshots or secrets.

The route grouping is proposed navigation, not an additional product specification. Collection/detail handlers can accept discriminated operations without multiplying one endpoint per action. Online forms and offline replay converge on the same application operations and replay rules. This allocation uses Route Handlers; optional Server Actions do not need a second parallel command implementation. Connect uses its own outbound runtime entry, so there is no `/api/inngest` callback route here.

Settings show safe approved configuration; this tree does not settle whether task mapping is edited in the app, a mounted configuration file or 9router administration. Provider credentials/OAuth/fallback administration stays in the private router's administration path. Edit/void/opening-balance policy, rejected-image lifetime, backend destinations, host image, IAM needs and resource budgets remain the open decisions described below. The named files are places to implement the chosen policy, not decisions on those policies.

Deferred chat, automatic agent approval, multi-user admission, shared workspaces, statement imports, native clients, full database replication, optional Ollama and a separate request API are outside this MVP allocation. Their absence is deliberate scope, not a directory left unfinished. The three Terraform roots remain the complete application root set; the proposed remote backend is bootstrapped separately through explicit operations, before AWS backend initialization. No automatic infrastructure deployment is implied by the proposed CI check file.

Feature-owned SQL stays at `modules/<feature>/adapters/`; shared external adapters and cross-feature transaction binding stay under `src/adapters/`. Both are outer code. This keeps SQL near its owner without allowing `application.ts` to import it. Avoid global `repositories/`, `services/`, `utils/` and `lib/` folders that scatter one behaviour across technical layers.

Consolidate shared database setup under `src/adapters/postgres/`, including the pool, Drizzle schemas and owner-context mechanics; do not create a separate `src/db/`. Group schema files by owning module. The database-specific name makes the dependency explicit. If another database is adopted, its shared implementation belongs under its own directory, such as `src/adapters/mysql/`, with corresponding feature adapters. Replacing a database still requires adapting queries, migrations and database-specific guarantees; core-owned ports and financial policy remain independent. Keep reviewed migration files in the root `drizzle/` directory and point migration tooling at the adapter-owned schemas.

Do not introduce microservices, event sourcing, a dependency-injection container or one repository per table. Double-entry records and append-only audit do not by themselves require event sourcing. A module may contain cohesive functions without an entity hierarchy.

## Ownership and interfaces for this app

Retain six server feature modules and the browser outbox. Add `money` as a small pure module because intake, ledger and browser capture share exact value semantics; it is not another deployment or bounded context. These are proposed responsibilities, not measured depth in existing code.

| Module | Behaviour hidden behind its interface | Outbound needs and limits |
|---|---|---|
| `money` | Exact IDR parsing/representation, currency checks, positive movement versus nonnegative balance/fee rules | No I/O/framework imports. Retain `IdrMinorUnits` and `parseIdrToMinorUnits`. Exact decimal/integer operations throughout. |
| `ledger` | Account/Category, intent → balanced Postings, atomic Transaction/AuditLog/replay outcome, reversal and reads | Ledger-owned storage/read contracts. No SQL handles; no imports from proposals/planning. |
| `proposals` | AgentProposal transitions, duplicate review, explicit approval/rejection and undo | Proposal storage and atomic approval-work contract; public ledger/retention capabilities. `propose_only` remains the MVP rule. |
| `intake` | Untrusted extraction validation, arithmetic checks, unresolved references | Existing `AiGatewayPort` and pure money. No ledger commit/approval capability. SDK/image encoding belongs in the AI adapter. |
| `planning` | BudgetPeriod/payday policy, RecurringRule/Occurrence uniqueness, Goal progress | Own storage/read contracts and ledger read/authorized posting capabilities. Recurrence permission differs from agent autonomy. |
| `insights` | Deterministic metrics, metric-reference narrative validation, fallback and InsightReport | Scoped projections, report storage and AI contract. Core code owns metric meaning even when SQL performs aggregation. |
| `retention` | Attachment metadata, recoverable file lifecycle, first-approval expiry, read denial and purge recovery | Attachment persistence and existing `BlobStoragePort`. No public filesystem paths or ledger imports. |
| `offline-outbox` | Durable owner-scoped commands, bounded captures, retry/acknowledgement, pending/error states | Queue-store contract implemented with IndexedDB; existing `OutboxTransportPort`. Browser events/canvas/service worker are adapters. |

Keep the graph acyclic: proposals → ledger/retention; planning → ledger; insights → ledger/planning. Intake returns validated interpretation data for proposal submission rather than importing proposal persistence. Workers may sequence extraction and submission, but financial approval policy stays inside proposals. Transaction wiring does not give ledger a dependency back on proposals.

`commitEntry` accepts expense/income/transfer intent and hides sign assignment, balancing and audit. `decideProposal` hides version checks, approval and first-approval expiry. Reject an interface that makes a route load rows, construct postings, append audit and mark the proposal itself; a short handler built from many exported primitives still leaks the workflow.

Specify owner scope, replay conflicts, version/locking expectations and whether an error guarantees no committed changes. Adapters and fakes must honour those semantics. A small conversion/security adapter can be worthwhile even when it has little depth; eliminating every thin function is not the goal.

## Atomic work without leaking Drizzle

Approval crosses proposals, ledger and retention. A repository per table cannot solve that transaction requirement. Recommend a narrow **application-owned atomic-work contract**, implemented by the PostgreSQL adapter, rather than a generic transaction object exposed to callers.

The approval contract belongs with proposals because that module owns the transition. Its callback receives only what it needs: locked/versioned proposal storage, transaction-bound ledger writing and attachment approval metadata. It exposes no raw SQL executor, arbitrary table access, `commit` or `rollback`. Standalone ledger commands use a smaller ledger-owned work scope; avoid a universal session containing every module.

For `decideProposal`, the proposed execution is:

1. The request adapter verifies session/provisioned owner, translates input and calls proposals. Jobs derive ownership from authoritative records, not an event's claimed owner.
2. Proposals requests atomic work. The PostgreSQL implementation begins one transaction, sets owner context on that connection and constructs transaction-bound capabilities. Lookup, locking and replay checks happen within this scope.
3. Proposals checks the pending version/decision, invokes ledger behaviour to validate intent and persist postings/audit/replay, then invokes retention behaviour to record first approval and expiry. Update the proposal outcome in the same transaction. Rejection creates no posting.
4. Failure aborts the scope. Only a successful SQL commit produces a durable acknowledgement; retry returns the original result or a documented conflict and cannot approve twice.

Composition binds feature factories to the same transaction; it does not decide which records approval changes. Bound ledger/retention operations join that scope and cannot independently commit or open new work. Reusing a standalone ledger operation must not secretly create a second transaction.

Define one abort convention before writing the adapter. Returning `{ ok: false }` is a normal callback return unless the work implementation explicitly converts it to rollback. Drizzle documents exception-based rollback and nested savepoints; a savepoint is not an independent financial commit. Translate known aborts to public errors after rollback and propagate unexpected failures. [Drizzle transactions](https://orm.drizzle.team/docs/transactions).

The core owns business atomicity; PostgreSQL implements it. Keep owner-scoped uniqueness, foreign keys and balancing protection in reviewed migrations as independent safeguards. A normal `CHECK` cannot enforce a sum over multiple Posting rows; select and test a deferred trigger or controlled write path. [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

RLS remains infrastructure: protected queries use verified owner context on the same transaction/connection. Auth lookup has scoped access before that context exists. Runtime roles must lack table ownership, superuser and `BYPASSRLS`; table owners normally bypass RLS. Verify missing/wrong owner and pooled-context isolation on real PostgreSQL. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

Exact TypeScript callback types, lock/isolation strategy and abort representation remain implementation work. This fixes ownership and dependency direction without pretending those contracts already exist.

## Reads, wire contracts and environments

**Use projection ports for simple reads.** A module's application read operation accepts validated filters and returns an owner-scoped read model. Its SQL adapter can aggregate/join without reconstructing entities or entering a command workflow. Server Components call the authorized server application interface directly; they need not call their own HTTP endpoint. Routes/UI never import Drizzle to bypass the seam. This preserves query efficiency without full CQRS. [Next.js backend guidance](https://nextjs.org/docs/app/guides/backend-for-frontend#server-components), [Fowler](https://martinfowler.com/bliki/CQRS.html).

**Wire contracts have a different lifetime from internal values.** `contracts/` owns versioned JSON-safe command/read schemas. Amounts cross the wire as integer decimal strings plus currency; core code reconstructs exact validated values. Preserve the existing project convention of 100 minor units per rupiah. No Drizzle types, session objects, file paths or model SDK objects cross the wire. Share pure values/schemas when semantics match; retain conversions for wire versions, money and persisted rows without requiring mapper classes.

**The browser is a separate trust and execution environment.** UI/outbox/browser composition may import wire contracts, pure money and browser-safe interfaces. They cannot import server feature entries, SQL, filesystem/AI implementations or secrets, including through barrels. Server validation runs again: schema shape does not establish authority. Keep stable HTTP command IDs and compatible replay across deployments; persisted commands must not use framework-generated Server Action references. [Offline contract](./tech-stack.md#4-offline-and-ai-boundaries), [Next.js action security](https://nextjs.org/docs/app/guides/data-security).

Next.js client imports bring dependencies into the client graph. Place `server-only` at web-facing integration/composition entries and check actual builds. Avoid that marker in framework-independent code shared with a standalone worker; enforce the worker environment through separate compilation/import checks. Worker composition cannot import Next.js session/request utilities. [Next.js environment guidance](https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning).

## Jobs, files and infrastructure

Clean Architecture changes source dependencies, not runtime durability. Preserve the latest notes' recovery requirements:

| Responsibility | Placement and behaviour |
|---|---|
| Receipt capture | Web calls retention. Validate bounded input, hash on the server, coordinate private file state with Attachment/dispatch records, and acknowledge only a durable recoverable capture. Temporary/missing/orphan files need reconciliation because SQL and filesystem writes are separate failure domains. |
| Dispatch | An independently supervised dispatcher claims committed pending records, publishes record IDs, records acknowledgement and retries with bounded backoff/reclaim. It starts and scans without Connect or the Inngest cron/event it is recovering. PostgreSQL is the recovery source; duplicate delivery remains possible. |
| Durable execution | Connect functions adapt workflow steps to application operations; they contain neither SQL nor financial transition policy. Persist IDs/status as step outputs; keep image bytes/full extraction snapshots out of durable history. Connect requires a long-running process and persistent outbound connection. [Inngest Connect](https://www.inngest.com/docs/durable-execution/deploying-functions/connect), [workflow checkpoints](https://www.inngest.com/docs/durable-execution/durable-workflows). |
| Reconciliation | Startup/periodic runner recovers stalled captures, due occurrences, missing reports and expired files independently of orchestration. Share application behaviour with scheduled functions. |
| Retention | First approval fixes expiry at approval + seven days in the financial transaction. Deny reads at expiry, retry physical deletion, preserve expiry across edits/retries/undo/restore. Keep bytes outside `public/`, caches, logs and long-lived backups. Physical deletion delay and abandoned/rejected-image policy remain open. |
| Infrastructure | Shared Terraform runtime module; separate `mac-runtime`, `aws-infra`, `aws-runtime` roots/state. Declare web, Connect and dispatcher/recovery supervision explicitly. Keep migrate/provision/restore as explicit `ops/` commands. [Existing operating contract](./tech-stack.md#6-terraform-on-mac-and-aws). |

AI remains untrusted inference behind `AiGatewayPort`; deterministic application operations own financial authority. Keep SDK/provider formatting in `ai-ninerouter` and revalidate extraction in intake. No model approval authority or paid fallback follows from restructuring. Private files remain behind `BlobStoragePort`; moving web/worker across hosts later requires an actual shared-storage implementation.

## Enforcement and verification

Import checks must distinguish **core files from adjacent adapters**. A blanket prohibition on all `modules/**` infrastructure imports would incorrectly reject the SQL adapters in this tree. A blanket ban on internals would also reject a module's own implementation. Use caller-aware checks.

| Check | Evidence required during implementation |
|---|---|
| Dependency direction | Core `domain.ts`, `application.ts`, `ports.ts` and internals cannot import Drizzle/pg, Next.js, Inngest, filesystem, SDKs, browser globals or concrete adapters. Pure modules stay platform-neutral. |
| Visibility | Callers use `index.ts`; adapters/composition may use the assembly contract. Own internals are allowed. Cross-feature private imports and cycles fail, including ORM type-only imports into core contracts. |
| Environments | Cover aliases, relative paths, re-exports and dynamic imports; run browser/worker checks and real builds. `no-restricted-imports` alone handles static imports, not dynamic ones. [ESLint](https://eslint.org/docs/latest/rules/no-restricted-imports). |
| Rule reliability | Fixtures prove core → adapter and client → server imports fail, while adapter → port, own-internal and intended cross-module public imports pass. These verify tooling, not runtime authorization. |

Test behaviour through supported module interfaces. Use hand-computed expectations, deterministic AI, a controllable clock and stateful storage/work fakes; do not mock every internal function. An atomic-work fake must stage/rollback changes and implement replay semantics, rather than merely calling its callback.

| Test group | What it establishes |
|---|---|
| Module behaviour | Ledger/reversal/replay, competing approvals/rejection, expiry, payday/short-month rules and deterministic insight fallback. A Rp500.000 source balance, Rp100.000 transfer and Rp1.000 fee leave Rp399.000 at source, add Rp100.000 at destination and consume only Rp1.000 of expense budget. |
| Adapter contracts | Run applicable storage/work scenarios against fakes and SQL adapters. Use PGlite for SQL/migration feedback where compatible; it cannot establish production roles, pooling, locks or concurrent connections. [PGlite](https://pglite.dev/docs/). |
| Database/filesystem guarantees | Real PostgreSQL verifies constraints, owner denial, concurrent replay/approval and rollback after later failure. Temporary-directory tests verify path/symlink/rename/deletion behaviour that a blob fake cannot prove. Direct SQL assertions are appropriate when SQL guarantees are the subject. |
| Browser/runtime | Offline save/replay/lost response, old payloads, quotas, owner switch/auth expiry; crash between file/commit/send/ack; recovery without orchestration; expiry before reads after restore. Record actual installed iPhone checks separately. |

Focused pure-policy tests are useful at meaningful interfaces; do not export private helpers solely for tests. Replace redundant implementation-coupled tests when stronger seam tests cover the behaviour while retaining distinct adapter/failure coverage. There are no application tests to remove yet.

## Implementation sequence and open choices

Start one manual-ledger vertical slice: wire input, owner verification, exact money, ledger behaviour, narrow storage/work contract, PostgreSQL adapter and migrations. Prove fee accounting, replay, rollback and RLS before expanding the catalogue. This tests the design against a real transaction instead of creating empty layers.

Add offline replay and version policy next. Then implement pending proposals and atomic approval with transaction-bound ledger/retention capabilities, followed by receipt/file recovery, durable dispatch and Connect extraction. Planning, insights and operational recovery follow. Develop policy and adapter integration together; neither a fake-only core nor a complete directory tree constitutes the app.

Choose port signatures from those slices. Reconsider a seam when callers still need its sequencing details, tests require large mock graphs or changing one rule touches many modules. Use the deletion test: if deletion spreads complexity among callers, the module hid useful behaviour; if it only removes forwarding, simplify it. Preserve deliberate conversion/security adapters even when small.

Open choices include lock/isolation and abort semantics, capture/dispatch recovery states, compatible wire versions, opening-balance/reversal policy and operating retention/restore targets. Money ownership is now recommended explicitly, but rounding/input edge cases still need agreement. AWS budgets, auth provisioning and multi-user scope remain open in the companion notes. Architecture patterns do not settle those product decisions.

## Research limits

Reviewed the guide, repository inventory and relevant sections of all four companion notes, and checked the original linked primary/official sources on 2026-10-31. Expanded the proposed tree to named file leaves and rechecked Next.js file conventions and Terraform module structure on 2026-11-01. No `.codegraph/` index exists, so CodeGraph was skipped. No application, ADR, glossary, dependency or infrastructure implementation was created; no runtime, database, provider or package-version compatibility check occurred. This revision changes only this research recommendation.
