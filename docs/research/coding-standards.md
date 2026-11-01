# Research: Coding Standards for Deep Modules with Clean Architecture

> Researched 2026-11-01 (Asia/Jakarta). This document proposes a coding contract for the tracker; it does not report existing enforcement. The repository currently contains contributor guidance, research and diagram artifacts, with no application source, package manifest, migrations, test suite or CI configuration. Recommendations follow all five existing research notes and current primary sources. No dependencies, configuration or infrastructure were installed or changed.

## Recommendation and scope

Make **Deep Modules and inward source dependencies** the governing coding standard. A feature should hide meaningful policy and workflow knowledge behind a convenient behavioural interface. Its domain and application code own the contracts they need; frameworks, SQL, files and model SDKs implement those contracts from outside. Review both qualities together: a convenient facade that imports Drizzle into financial policy fails the dependency rule; a dependency diagram filled with forwarding interfaces fails the information-hiding goal.

Keep the proposed feature-first modular monolith, Next.js request application and separate Inngest Connect worker. Use strict TypeScript, one formatter, type-aware ESLint, explicit environment checks and meaningful financial/recovery tests to support that design. Readable names and ordinary control flow matter at every level. A reviewer should locate a change's owner and understand its result without following a chain of forwarding functions. This applies the latest [codebase structure proposal](./codebase-structure.md#module-shape-and-dependency-direction); Google's review guidance likewise asks reviewers to assess unnecessary complexity. [Google engineering practices](https://google.github.io/eng-practices/review/reviewer/looking-for.html).

Rules below are **project recommendations** derived from those requirements. Official sources establish language and tool behaviour; they do not prescribe this project's folder names, exact naming style or test allocation. Within this proposal, “require” describes the recommended acceptance standard once adopted.

### Daily baseline

- Name things by domain meaning and units; hide related rules and transition sequencing with their owner.
- Keep source dependencies inward; add a narrow core-owned port when independence requires it, even with one production adapter.
- Use cohesive functions and straightforward control flow; create another layer when it supplies a distinct abstraction.
- Parse untrusted data, use explicit state alternatives, and keep mutation owned and local.
- Protect exact money, verified owner scope, atomic changes, replay and fixed receipt expiry at the application seam.
- Keep browser, server and worker imports within their explicit environments.
- Run the shared checks and relevant behaviour tests; document actual results and unresolved assumptions.

Start with [the design standard](#deep-modules-and-clean-architecture-in-daily-code) and [decision criteria](#when-to-deepen-split-add-a-port-or-keep-a-conversion). Use [everyday readability](#everyday-readability), [TypeScript contracts](#typescript-contracts-and-validation), [module ownership](#module-ownership-and-environment-boundaries) and [delivery gates](#formatting-lint-and-delivery-gates) for implementation. The [short review checklist](#short-review-checklist) fits a normal change review.

### Coverage of the existing research

| Local source, read in full | Requirements carried into these standards | Conflict or limitation retained |
|---|---|---|
| [Agentic financial tracker](./agentic-financial-tracker.md) | IDR, double-entry Postings, offline capture, payday periods, `propose_only`, seven-day image retention, accessible mobile UI | Historical package versions, Supabase and hosted-service recommendations do not define the current implementation. Confidence thresholds and “about 50 approvals” do not establish safe automation. |
| [Architecture and system design](./architecture-and-system-design.md) | Intent-based financial operations, named feature owners, AI/storage/transport contracts, worked transfer example and behaviour tests | Its Drizzle-in-core examples, parser and absolute testing rules need the corrections below. Its runtime timings are not measured acceptance targets. |
| [Tech stack](./tech-stack.md) | No Supabase, self-hosted Inngest, proposed PostgreSQL/Drizzle, Better Auth, filesystem, `idb`, Next.js/Connect, Terraform on Mac/AWS | Replacement components remain proposals; supported package compatibility, owner provisioning, deployment and cost still need evidence. |
| [App architecture](./app-architecture.md) | Durable dispatch, file/SQL recovery, owner verification, concurrent approval, exact values, stale/offline UI, retention across copies | Its concrete Drizzle transaction placement yields to the newer structure note; its failure-handling requirements remain relevant. |
| [Codebase structure](./codebase-structure.md) | Core-owned persistence/atomic-work contracts, pure `money` module, feature-adjacent SQL adapters, separate browser/server/worker composition, independent dispatch/recovery | The tree is a proposed allocation, not a demand to scaffold every file. Exact port signatures, locking and abort representation remain open. |

Use [tech-stack.md](./tech-stack.md#1-recommendation) for the newer infrastructure direction and [codebase-structure.md](./codebase-structure.md#recommendation-and-evidence) for the newer source-dependency direction. Preserve product requirements unless a later adopted decision changes them. Follow [the repository's domain guidance](../agents/domain.md) for terminology and ADRs, and [its issue tracker guidance](../agents/issue-tracker.md) for GitHub issue work. This research does not promote any unadopted proposal into an ADR.

### Priority and enforcement

| Kind | Examples | How to assess |
|---|---|---|
| Deterministic checks | Formatting, type errors, unused symbols, unsafe promises, forbidden imports | Compiler, formatter, lint and tested import checks; fail CI once configured. |
| Behavioural requirements | Balanced Postings, owner denial, rollback, safe replay, expiry, no unapproved posting | Meaningful module, integration and browser tests, plus review of the invariant. |
| Design judgment | Cohesion, names, useful comments, when to extract a helper or contract | Review against concrete callers and change scenarios; don't invent line-count gates. |
| Release evidence | Real PostgreSQL roles/concurrency, installed iPhone behaviour, backup restore, physical deletion delay | Explicit checks against the deployed environment; a passing unit suite cannot substitute. |

A formatter cannot prove maintainability, and a branded type cannot prove authorization. Keep each gate honest about what it establishes.

## Deep Modules and Clean Architecture in daily code

Ousterhout's module interface includes behaviour, side effects and usage constraints as well as signatures. He also warns about splitting code by execution stage and adding layers that repeat the same abstraction. For this standard, judge depth by useful functionality relative to caller knowledge, rather than implementation line count or the number of exported functions. [John Ousterhout, Stanford CS190: Modular Design](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php?topic=modularDesign).

Martin's dependency rule concerns **source references**. Runtime control can call outward through an inward-owned contract. His circles are schematic, and entities may be data with functions. Framework-generated data shapes must stay outside the core. [Robert C. Martin, The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). Cockburn's original ports-and-adapters article similarly makes application conversations independent of their external devices and supports driving the application through different adapters. [Alistair Cockburn, Hexagonal Architecture (2005)](https://alistair.cockburn.us/hexagonal-architecture).

Distinguish that architectural **interface** from TypeScript's `interface` keyword and from an injected outbound **port**. Pure `money` has a meaningful behavioural interface even with one implementation and no injected port. Application ports here mostly protect external capabilities; counting adapters does not establish whether their contract is useful.

For this tracker, apply those principles as follows. These are project judgments, with placement defined in [codebase-structure.md](./codebase-structure.md):

- **Own a coherent decision.** `commitEntry` accepts expense/income/transfer intent and owns reference validation, signs, balancing, audit and replay. `decideProposal` owns the proposal transition and atomic collaboration with ledger and retention. Keep those rules out of routes, components, job functions and composition.
- **Hide knowledge, retain necessary promises.** The caller should not construct signed Postings or know table-write order. It must know verified owner requirements, expected conflicts, durable acknowledgement and retry semantics. Hiding failure semantics makes an interface harder to use safely.
- **Keep one behavioural seam with useful internals.** A feature may contain posting construction, replay policy and persistence mapping files. Export supported operations/types through `index.ts`, expose assembly contracts deliberately through `ports.ts`, and allow the feature's own internal imports. A single seam does not require a single huge file.
- **Keep policy independent.** Core source, including type-only imports, cannot reference Drizzle/pg, SQL row types, Next.js, Inngest, filesystem or provider implementations. Outer composition constructs adapters and injects core-owned capabilities. Feature-adjacent SQL is still outer code.
- **Use the simplest expression of the rule.** Pure functions and immutable values suit money, posting construction and budget dates. Choose a stateful object when it improves lifecycle encapsulation. Do not require rich entity classes, four folders per feature, a matching interface for each operation, CQRS or event sourcing. An append-only audit and double-entry ledger do not imply event sourcing.

### Assessment of the supplied guide

The user-supplied `deep_modules_and_clean_architecture_guide.md` was read in full as a secondary reference, not contributor instructions. Its main claims were checked against the original authors. The following assessment adapts its examples and decision matrix to this app.

| Guide claim | Decision | Standard for this app |
|---|---|---|
| A deep module has a tiny interface hiding substantial work. | **Keep, refine the measure.** | Count caller knowledge, including ordering/error obligations. A facade exposing raw SQL or mandatory caller-side balancing remains leaky even with one method. [Ousterhout](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php?topic=modularDesign). |
| Clean Architecture favours many small classes and strict layer ceremony. | **Correct.** | Inward dependencies govern the design; a prescribed class/folder count does not follow. Use additional layers only for a distinct responsibility. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |
| Introduce interfaces only for I/O or multiple implementations. | **Useful port heuristic, not a universal definition.** | The app's financial storage/work ports do protect I/O and are justified with one production adapter. Pure `money` still has a behavioural interface; ordinary functions need no mirrored TypeScript interface. [Cockburn](https://alistair.cockburn.us/hexagonal-architecture), [Ousterhout](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php?topic=modularDesign). |
| Business rules must live in rich entities rather than procedural use cases. | **Reject mandatory rich OOP; keep cohesive rule ownership.** | Pure data plus functions can own rules and valid transitions. A thin orchestrator is useful when it coordinates meaningful policies. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |
| Feature packaging enforces private implementation; layer folders force public code. | **Correct.** | Feature folders improve discoverability. TypeScript file exports and import checks establish access; directory placement alone supplies no package-private protection. [TypeScript modules](https://www.typescriptlang.org/docs/handbook/2/modules.html). |
| Simple reads may bypass domain entities. | **Keep with authorization and dependency limits.** | Authorized application reads use core-owned projection contracts; SQL adapters may join/aggregate directly. Routes do not acquire SQL access and policy metrics retain their defined meaning. Separate read shapes require no separate database or command bus. [Local read design](./codebase-structure.md#reads-wire-contracts-and-environments). |
| Identical DTO fields make mapping redundant. | **Qualify.** | Reuse core-owned shapes when meaning, trust and lifecycle match. Keep conversions for wire versions, exact money, owner verification and ORM rows even when today's fields look identical. [Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). |
| The concise checkout example demonstrates the combined design. | **Useful encapsulation; incomplete for this app's failure model.** | Charge followed by save has a partial-success gap. Here, file writes, SQL commit and event publication likewise need recorded recovery and replay. A small interface cannot supply cross-resource atomicity. [Local failure flows](./app-architecture.md#receipt--proposal--approved-ledger-change), [Stripe on unknown outcomes](https://docs.stripe.com/error-low-level). |

### When to deepen, split, add a port or keep a conversion

Use this decision process during implementation and review. No line-count, number-of-callers or two-adapter threshold replaces the stated reason.

| Decision | Evidence that warrants it | Tracker example and limit |
|---|---|---|
| **Deepen or consolidate** | Callers repeat a rule, coordinate a fragile sequence or must understand internals to perform one intent. | Move signs, balancing and replay into `commitEntry`; move approval/ledger/expiry coordination into `decideProposal`. Combine responsibilities only when they share knowledge. |
| **Split internally** | One implementation contains independently understandable policies or different reasons to change, and a named internal contract reduces the reader's context. | Separate exact posting construction from replay coordination without exporting both as caller responsibilities. Prefer a private function/file before a public feature. |
| **Create a separate module** | A cohesive capability has its own meaningful vocabulary and contract, and callers can use it without learning its former owner. | Pure `money` is shared by browser, intake and ledger; avoid making intake the dependency for all money semantics. Do not create a shared module from coincidental syntax. |
| **Add a core-owned port** | Policy otherwise imports an external detail, or a meaningful capability needs explicit substitution/control. State required semantics, not driver methods. | Ledger storage/read and proposal atomic-work contracts are warranted now. Inject a clock/ID function where needed; avoid a universal `DatabasePort` or one repository per table. |
| **Keep a direct internal call** | Both sides share policy ownership; no external detail or independent capability needs hiding. | `buildPostings(intent)` need not acquire `PostingBuilderInterface` and `PostingBuilderImpl`. Do not add a port merely to spy on the call in tests. |
| **Keep a conversion/security adapter** | Trust, representation, versioning, environment or authority changes even if field names match. | HTTP money string → validated `bigint`; session → owner; row → core value; v1 persisted command → current input. Small useful adapters need not be deep. |
| **Use a projection read** | The operation supplies authorized display data without changing state or deciding a financial transition. | `listLedgerActivity` can read a scoped SQL projection; do not rebuild a Transaction entity solely to display its merchant. Keep money/metric semantics and owner checks intact. |

Before adding an abstraction, state what knowledge it hides and which source dependency it protects. Before deleting one, inline it mentally: does that spread policy/ordering among callers, remove necessary security/conversion, or simply remove forwarding? After a change, trace one normal call and one failure. If a caller still must understand write order, a mock graph reproduces the whole implementation, or one policy change requires editing several callers, reconsider the seam.

## Everyday readability

### Names that reveal meaning

Use the same term for the same financial concept. `Transaction`, `Posting`, `AgentProposal`, `Attachment` and `BudgetPeriod` come from the local domain notes; don't alternate between `Transaction`, `record`, `item` and `entry` for the entity. Existing operation names such as `commitEntry` can remain stable without renaming the entity. Prefer a verb that states the operation: `decideProposal`, `resolveBudgetPeriod`, `parseIdrToMinorUnits`, `listLedgerActivity`.

Propose these conventions for authored TypeScript:

| Item | Convention | Example |
|---|---|---|
| Variables, functions and parameters | `camelCase`, meaningful in local context | `approvedAt`, `loadPendingProposal` |
| Types, interfaces and React components | `PascalCase`; no `I` prefix for interfaces | `ProposalDecision`, `LedgerStore`, `ProposalReview` |
| Ordinary files and directories | `kebab-case`; names describe responsibility | `expiry-policy.ts`, `proposal-review.tsx` |
| Framework files | Keep required framework conventions | `page.tsx`, `route.ts`, `layout.tsx` |
| Database columns | `snake_case`; translate at the adapter boundary | `approved_at` → `approvedAt` |
| Booleans | State or capability, with positive wording | `isExpired`, `canRetry`, `hasPendingCommands` |
| Values with units | Include the unit or representation when ambiguity matters | `amountMinor`, `timeoutMs`, `byteLength`, `occurredAt`, `localDate` |

Prefer named exports for application code so search finds the same name at definition and use. Use default exports when the framework requires them. Keep ordinary imports explicit; reserve `index.ts` for supported feature entry points and deliberate registries, rather than adding barrels in every directory. These are local consistency choices. Next.js owns its special file conventions. [Next.js file conventions](https://nextjs.org/docs/app/api-reference/file-conventions).

Names such as `data`, `manager`, `process` and `utils` need enough context to identify their responsibility. A short `row` inside a SQL mapping loop is clear; a public `processData()` is not. Put IDR parsing in `modules/money`, expiry in retention and query keys in browser query ownership. Extract a shared abstraction when callers share meaning and lifecycle; similar syntax alone isn't enough.

### Functions and control flow

Keep a function focused on one understandable operation at its level. An approval workflow can legitimately coordinate several steps because approval is its responsibility; each step should have a domain name and an explicit outcome. Extract a helper when it hides a coherent rule, has a useful independent contract, or removes repeated knowledge. Don't split code merely to make every function short.

Use guard clauses for invalid or exceptional states, followed by the normal path. Prefer explicit intermediate values over nested ternaries, nested callbacks and long chains that combine parsing, policy, I/O and logging. A `for...of` loop is often clearer than an asynchronous `reduce`. Avoid boolean arguments such as `save(proposal, true, false)`; use a discriminated operation or a named options object when the choices matter.

Keep dependent work sequential. Run independent I/O concurrently only when its failure, cancellation and resource behaviour are clear; unbounded `Promise.all` over uploads, DB commands or AI calls can exceed the host or provider budget. No fixed maximum function length, file length or parameter count is proposed. Review whether a reader must hold unrelated details in mind. [Review rationale: Google complexity guidance](https://google.github.io/eng-practices/review/reviewer/looking-for.html#complexity).

### Mutation and comments

Default to `const`, readonly input properties and readonly collections at module seams. Avoid mutating caller-owned objects, React props/state or shared singleton data. Local mutation in a parser, accumulator or adapter is acceptable when ownership is clear and the result doesn't expose mutable internal state. TypeScript `readonly` is a compile-time restriction and is shallow; it does not freeze an object or prevent mutation through another alias. [TypeScript object types](https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties).

Comments should explain a constraint, surprising choice or failure mode. For example, “Keep the original expiry during undo; approval fixes the image deadline” explains a reason. “Set expiry” repeats code. Document public operations' owner scope, side effects, replay/conflict behaviour and error/rollback guarantees; avoid JSDoc on every obvious local helper. Link unusual workarounds to an issue or owning source, remove obsolete comments with the code, and give actionable TODOs an issue reference. [Review rationale: Google comments guidance](https://google.github.io/eng-practices/review/reviewer/looking-for.html#comments).

## TypeScript contracts and validation

### Compiler baseline

Recommend `strict: true` from the first source file, plus explicit `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitReturns`, `noFallthroughCasesInSwitch` and `forceConsistentCasingInFileNames`. The first two need separate opt-in; don't assume `strict` enables them. `noUncheckedIndexedAccess` makes potentially missing indexed values visible, while `exactOptionalPropertyTypes` distinguishes an absent property from explicitly setting it to `undefined`. [TypeScript strict](https://www.typescriptlang.org/tsconfig/strict.html), [indexed access](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html), [optional properties](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html).

Keep module-resolution, JSX, target and emit settings compatible with the selected Next.js and standalone worker tooling. Don't copy one generic tsconfig into every environment. Browser checking should not silently acquire Node globals, and a successful Next.js build does not prove that a standalone Connect artifact starts correctly. Use the separate environment checks proposed in [codebase-structure.md](./codebase-structure.md#reads-wire-contracts-and-environments).

### Parse unknown input; preserve useful inference

Treat HTTP bodies, persisted offline records, job payloads, configuration and model output as `unknown` until parsed. Use Zod for wire and external schemas, then let deterministic domain constructors validate meaning. Derive a wire TypeScript type from its schema where they describe the same contract; keep domain values separate when wire lifetime, money representation or authority differ. Zod's `safeParse` returns a discriminated success/error result. [Zod basic usage](https://zod.dev/basics).

Avoid `any`, `as any`, non-null assertions and casts that bypass a check. A narrow assertion can be reasonable inside a validated brand constructor or a documented third-party adapter limitation; contain it there and test the boundary. Don't spread exceptions throughout callers. Public application operations and ports should declare input/result types; infer obvious local variables and private helper returns rather than restating the compiler's work.

Use `null` when a stored/domain field explicitly has no value; use an optional property when an input can omit a change. For an edit, distinguish “leave merchant unchanged” from “clear merchant” in the contract. Don't coerce missing or invalid amounts to zero. IDs with different meanings merit separate branded types where accidental interchange is plausible, but a brand supplies neither runtime validation nor permission.

### Make states explicit

Model alternatives with discriminated unions rather than combinations of optional fields and booleans. An acknowledged command requires a resulting ID; a paused command should say why. Handle each supported alternative explicitly, with exhaustiveness checked by TypeScript or a typed lint rule. [TypeScript narrowing and `never`](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking), [typescript-eslint exhaustive switches](https://typescript-eslint.io/rules/switch-exhaustiveness-check/).

Illustrative standalone contract; these types do not add implementation or settle HTTP status codes:

```ts
type CommandOutcome =
  | { readonly status: 'acknowledged'; readonly transactionId: string }
  | { readonly status: 'paused'; readonly reason: 'authentication_required' }
  | { readonly status: 'rejected'; readonly code: 'invalid_input' | 'conflict' }
  | { readonly status: 'retryable'; readonly retryAfterMs: number };

function assertNever(value: never): never {
  throw new Error('Unsupported command outcome', { cause: value });
}

function commandStatusText(outcome: CommandOutcome): string {
  switch (outcome.status) {
    case 'acknowledged':
      return 'Saved';
    case 'paused':
      return 'Sign in to continue';
    case 'rejected':
      return outcome.code === 'conflict' ? 'Review the conflict' : 'Edit the input';
    case 'retryable':
      return 'Waiting to retry';
    default:
      return assertNever(outcome);
  }
}
```

Use the selected exhaustiveness style consistently. Validate an external status before it reaches this typed function; keep unexpected error causes inside the application rather than sending them in public responses. Exhaustiveness only covers the variants known to the compiler.

## Module ownership and environment boundaries

Use the latest [proposed structure](./codebase-structure.md#ownership-and-interfaces-for-this-app). Each feature exposes behaviour through `index.ts`; core-owned `ports.ts` serves adapters/composition. Keep those entries distinct: importing the feature must not construct a pool or re-export its SQL adapter. Core application workflows may perform effects through injected ports; only pure domain policy must be effect-free.

The permitted source direction is request/job adapter → application → domain/own ports, with concrete adapters → owning ports/types. Composition imports factories and adapters and connects them. Runtime application → adapter calls occur through those ports; they do not authorize core → adapter source imports. Keep browser, web and worker composition explicit rather than using one barrel that exports every environment.

| Code | May depend on | Keep out |
|---|---|---|
| Pure `money` and domain policy | Pure values and approved pure domain dependencies | Frameworks, I/O, environment variables, time/global randomness and SDK types |
| Feature application code | Own policy/ports, permitted public feature operations, injected clock/ID capabilities where needed | Drizzle/pg, Next.js, Inngest, filesystem and concrete adapters, including type-only imports |
| Feature SQL adapter | Its owning port/types, shared PostgreSQL setup | Business decisions moved from application policy into SQL mapping glue |
| Web composition/request adapters | Session verification, contracts, application factories, concrete dependencies | A second implementation of financial rules |
| Browser/UI/outbox composition | Browser-safe contracts, pure money, browser outbox interface/adapters | Server feature entries, SQL/filesystem/AI credentials, server composition |
| Worker composition/functions | Application interfaces and worker-safe adapters, Inngest at the edge | Next.js request/session utilities and browser globals |

Adapters may sit under `modules/<feature>/adapters/`; they remain outer code. Don't prohibit all infrastructure imports under `modules/**`, or prohibit a feature from using its own internals. Cross-feature callers use supported entries. Keep the dependency graph acyclic: proposals → ledger/retention, planning → ledger, insights → ledger/planning; intake cannot approve or commit financial changes.

Use plain factory functions for dependency injection and group related capabilities by the conversation the core needs. For example, a ledger storage contract may load reference values and persist a validated Transaction with its audit/replay result. It must not expose Drizzle builders under renamed `select`/`insert` methods. Specify owner scoping, concurrency, rollback and replay behaviour alongside types. Port independence protects policy; replacing PostgreSQL still requires new queries, migrations and equivalent state guarantees. It does not make a database swap cheap. [Local atomic-work design](./codebase-structure.md#atomic-work-without-leaking-drizzle).

### Behavioural calls and efficient reads

| Caller problem | Recommended interaction | Knowledge retained by the owner |
|---|---|---|
| Manual transfer or offline replay | Verify owner, parse command, call ledger `commitEntry` with positive principal, nonnegative fee and stable command ID. | Ledger validates references and creates signed balanced Postings, audit and replay result. HTTP/outbox handles the documented acknowledgement, conflict or unknown outcome. |
| User approves a pending proposal | Verify owner, parse decision plus expected version, call `decideProposal` once. | Proposals coordinates the transition, transaction-bound ledger behaviour and first-approval expiry. The route does not update three tables or reconstruct that ordering. |
| Dashboard needs balances/activity | Verify owner, validate bounded filters, call the application's read operation. | A core-owned projection contract returns the required read model; the SQL adapter aggregates/joins directly. No entity reconstruction, command pipeline or second read database is required. |
| Browser captures a receipt | Enqueue through outbox; upload through its transport; show local/server acknowledgement separately. | Outbox owns durable local enqueue/replay; retention owns recoverable server file/metadata/dispatch state. A short HTTP success path must not hide a lost file or unsent event. |

Simple read operations can be small because they provide authorization, projection and a stable contract. Avoid manufacturing a deep domain object for every read or deleting an authorized application read because it delegates. Return only needed data; metric definitions stay owned by ledger/planning/insights even when SQL computes aggregates. These are the [current read and environment recommendations](./codebase-structure.md#reads-wire-contracts-and-environments), consistent with Next.js's server-only, authorized data-access guidance. [Next.js data security](https://nextjs.org/docs/app/guides/data-security).

Enforce resolved imports, not just text patterns: aliases, relative paths, re-exports, type-only imports and literal dynamic imports can all cross a forbidden boundary. Restrict nonliteral dynamic loading to explicit runtime integration code. Give the checker positive and negative fixtures, including allowed own-internal and adapter→port imports. ESLint's `no-restricted-imports` only checks static imports; supplement it with a suitable resolved checker as the structure note proposes. [ESLint rule scope](https://eslint.org/docs/latest/rules/no-restricted-imports).

## Financial values, dates and state changes

Keep exact money in the pure `money` module. Use `bigint` minor units with currency, preserving the existing 100-minor-units-per-rupiah convention. Parse localized separators and shorthand with decimal/integer operations; don't route monetary arithmetic through `Number`, `parseFloat` or `Math.round`. Wire values use validated integer decimal strings and currency, while postings have signed amounts. Check PostgreSQL `bigint` range before persistence because JavaScript `bigint` itself has no corresponding fixed 64-bit range. PostgreSQL documents exact integer/numeric types separately from inexact floating-point types. [PostgreSQL numeric types](https://www.postgresql.org/docs/current/datatype-numeric.html).

Separate positive movement amounts, nonnegative fees/opening balances and signed postings/balances. A zero opening balance must remain valid. Do not blindly apply nonnegative rules to computed balances; account types can have different negative-balance semantics. Reject unsupported fractional precision or define an explicit rounding policy before accepting it. Formatting is presentation; charts must not calculate authoritative totals from floating-point display values. These are requirements derived from the [money proposal](./codebase-structure.md#ownership-and-interfaces-for-this-app), with account/rounding policy still open.

Worked expectations should remain easy to inspect:

| Behaviour | Expected result |
|---|---|
| Parse `Rp 45.000` | `4_500_000n` minor units |
| Parse `1,5jt` | `150_000_000n` minor units |
| Open an account at Rp0 | Valid zero opening balance |
| Rp500.000 source, transfer Rp100.000, fee Rp1.000 | Source Rp399.000, destination increases Rp100.000; expense budget increases only Rp1.000 |
| Posted Transaction | At least two Postings, summing to zero per currency |
| Same owner/command ID and same payload | Original committed outcome, no extra Postings or audit |
| Same owner/command ID and different payload | Explicit conflict, no new financial write |

Keep an instant, a Jakarta local date and a recurring local time distinct in names and types. Store RRULE, DTSTART and IANA timezone separately. Pass a clock or an explicit `asOf` value into time-sensitive policy so tests can cover month boundaries and expiry without sleeping. Preserve the agreed inclusive/exclusive budget-period convention at public seams; don't silently switch representations between modules. RFC 5545 ignores invalid recurrence dates; day 31 does not mean the last day of every month. [RFC 5545 §3.3.10](https://www.rfc-editor.org/rfc/rfc5545#section-3.3.10).

## Transactions, errors and asynchronous work

### Atomicity is part of the contract

Approval must commit its proposal outcome, balanced ledger records, audit/replay result and first-approval expiry together. A narrow proposal-owned atomic-work contract supplies locked/versioned proposal storage and transaction-bound ledger/retention capabilities. It exposes no SQL executor, arbitrary tables or public `commit`/`rollback`. The PostgreSQL adapter binds those capabilities to one transaction/connection; composition wires them without deciding the financial transition. A standalone ledger command uses its smaller ledger-owned scope. [Local atomic-work contract](./codebase-structure.md#atomic-work-without-leaking-drizzle), [node-postgres transaction connection requirement](https://node-postgres.com/features/transactions).

For example, after ledger writes succeed inside approval, a failure recording first-approval expiry must abort the proposal outcome, ledger, audit and replay records too. Bound feature operations join that existing scope and cannot independently commit or start another transaction. Cross-feature coordination must preserve this SQL-state guarantee. Keep network/model/file operations outside that SQL transaction and use durable recovery records for cross-resource workflows. The contract expresses required behaviour; actual SQL, locking and crash tests establish that the adapter delivers it.

Choose one explicit abort convention and test it before reusing the transaction adapter. Returning `{ ok: false }` from a Drizzle transaction callback is a normal return unless the adapter translates it into rollback. Drizzle documents exception-based rollback; nested transactions use savepoints. A later attachment/proposal failure must roll back earlier ledger writes, with tests proving the outcome through both fake and real SQL implementations. [Drizzle transactions](https://orm.drizzle.team/docs/transactions).

Use foreign keys, owner-scoped uniqueness and row checks alongside application rules. A plain `CHECK` cannot enforce a sum across other Posting rows; select and review a deferred constraint trigger or controlled write path. Model approval races and replay with database-backed concurrency control, rather than a process-local flag. The exact lock/isolation strategy remains implementation work. [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

### Expected rejection and unexpected failure

Use discriminated result errors for expected domain outcomes such as an archived account, stale proposal version, same-account transfer or conflicting command. Unexpected infrastructure/programming failures should propagate with their cause; an adapter can classify known database/provider errors into a stable application failure. Don't convert every exception into a generic success-shaped fallback. Node supports error causes when adding context. [Node error documentation](https://nodejs.org/api/errors.html#new-errormessage-options).

Translate failures at the request/job edge into a documented public contract. Distinguish authentication pause, permanent validation rejection, version/idempotency conflict and retryable infrastructure failure. Preserve an unknown commit outcome for safe replay; a network timeout does not prove the server rolled back. Don't expose SQL text, paths, tokens, stack traces or raw model output in user responses.

Catch only where code can recover, classify or add useful context. Log the unexpected failure once at the boundary responsible for reporting it, rather than logging and rethrowing at every layer. Use `finally`/bounded shutdown for owned resources and release transaction clients. Never use an empty catch to make a test or background job appear successful.

Await or return asynchronous work whose completion affects acknowledgement. A deliberately detached browser task still needs a rejection handler and visible failure state where applicable. `void flushOutbox()` can silence `no-floating-promises`; it does not handle rejection, and it cannot provide durable execution after a process exits. [typescript-eslint promise rule and its `void` caveat](https://typescript-eslint.io/rules/no-floating-promises/).

### Durable workflows and replay

Inngest entry points should adapt event/step lifecycle to application operations. Use stable, descriptive step identifiers; bound retries, concurrency and timeouts. Treat event delivery and external effects as repeatable, with database idempotency as the final guard. Inngest checkpoints completed step results and retries failed work; that doesn't make a filesystem write and SQL commit atomic. [Inngest durable workflows](https://www.inngest.com/docs/durable-execution/durable-workflows).

Send authoritative record IDs in events and return IDs/status from steps. Fetch and process receipt bytes within work that persists its validated result; don't checkpoint the bytes or a full sensitive extraction as a step return. Keep dispatch and reconciliation able to scan application records without needing the scheduler/Connect event that they are recovering. Preserve the separate supervised processes proposed in [codebase-structure.md](./codebase-structure.md#jobs-files-and-infrastructure).

Validate web/worker configuration at startup, construct pools/clients in composition and shut them down explicitly. Select a supported Node LTS satisfying all chosen dependencies, then pin it. Build and smoke-start the Connect artifact separately; if bundled, follow Inngest's documented SDK externalization requirements rather than assuming the web bundler handles it. [Node release policy](https://nodejs.org/en/about/previous-releases), [Inngest Connect bundling](https://www.inngest.com/docs/durable-execution/deploying-functions/connect#bundling-typescript-and-javascript-workers).

## React, Next.js and offline UI

Keep state near its owner: local React state for transient interaction, TanStack Query for server reads, and the explicit outbox for pending durable commands. Don't maintain a second copied balance or proposal state in a global store. Derive display values from current inputs during render; submit user commands from event handlers. Effects synchronize with external systems such as subscriptions or browser lifecycle listeners and must clean up those subscriptions. [React unnecessary Effects](https://react.dev/learn/you-might-not-need-an-effect).

Render must stay free of writes, uploads, approval and other side effects; don't mutate props/state. A render can run repeatedly. Start with straightforward components, add memoization when measurement shows it helps, and keep authoritative money/budget calculations outside charts. [React purity rules](https://react.dev/reference/rules/components-and-hooks-must-be-pure).

Use Server Components for server rendering/authorized reads and small Client Component boundaries where browser state or interaction requires them. Return minimal JSON-safe view models. Keep `server-only` at web integration/composition entries, while worker-shared core code remains framework independent. Check transitive client imports and actual output builds. Next.js documents how client boundaries pull dependencies into the client graph. [Next.js server/client components](https://nextjs.org/docs/app/getting-started/server-and-client-components).

Server Components call authorized application reads directly. Use stable versioned HTTP Route Handlers for offline replay; don't persist generated Server Action references in IndexedDB. Treat every Route Handler/Action as its own entry point for authentication, schema validation and resource authorization. A protected page or hidden button supplies no authority for a separate POST request. [Next.js data security](https://nextjs.org/docs/app/guides/data-security), [local replay design](./tech-stack.md#1-recommendation).

Give query keys every input that changes the result, including owner UUID, period and relevant filters. For example, `['budget-status', ownerId, periodStart]` avoids sharing one owner's snapshot with another. Persist only selected read snapshots with version/age metadata, and clear them on identity change. Keep pending commands in their separately owned store. Query keys determine cache identity; owner separation remains an application responsibility. [TanStack Query keys](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys).

Only claim “queued” after IndexedDB transaction completion, and remove a queued command only after durable server acknowledgement. Don't await HTTP/image processing inside an open IndexedDB transaction; `idb` documents that unrelated awaited work can let it auto-close. Keep schema upgrades/versioned old-command fixtures and handle quota/write errors visibly. [idb transaction lifetime](https://github.com/jakearchibald/idb#transaction-lifetime), [local offline requirements](./app-architecture.md#manual-or-offline-financial-command).

Show loading, empty, stale, queued, authentication-paused, conflict and retryable failure states where they affect the action. “Saved locally” and “posted to the ledger” require different evidence. Provide labels, keyboard access, visible focus, appropriate semantic controls and an accessible status message; keep reduced-motion support and avoid using colour as the only status signal. Treat these as implementation requirements from [the product UI notes](./agentic-financial-tracker.md#10-uiux--animation), with browser tests for the relevant interactions.

## Authorization, AI and sensitive data

Resolve Better Auth session → provisioned financial owner on each protected operation. Use that verified scope for record lookup, commands and attachment reads; a body/event owner ID is only an untrusted claim. Preserve the latest note's fixed-owner MVP while keeping owner scoping explicit. Better Auth documents server-side session lookup, while the financial permission mapping belongs to this application. Keep optional session cookie caching disabled initially as the [tech-stack proposal](./tech-stack.md#3-data-correctness-and-authorization) recommends. [Better Auth Next.js integration](https://better-auth.com/docs/integrations/next), [session management](https://better-auth.com/docs/concepts/session-management).

Test RLS with the actual non-owner runtime role and transaction-local context on the same pooled connection. Missing/wrong context must deny access. Table owners and roles with `BYPASSRLS` can bypass policies, so a successful query as a migration role proves little about application isolation. Keep auth-table lookup narrowly scoped before financial context exists. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

AI extraction has no tools and produces pending `AgentProposal` data. Deterministic code validates the schema, exact money, account/category ownership, dates, duplicates and permitted transition. The authenticated decision endpoint owns approval. Record model/prompt version and validation outcome without treating self-reported confidence as a calibrated probability. Reports derive their metrics deterministically and validate metric references/attribution before rendering a narrative. These follow the newer [agent authority proposal](./app-architecture.md#agent-authority-and-model-boundary).

Keep provider SDK details behind `AiGatewayPort`. Contract-test the selected 9router release/Combo for vision, schema handling, timeout/fallback and returned model identity. An OpenAI-compatible endpoint does not establish that every provider supports every multimodal or structured-output capability. Restrict task targets and prohibit unapproved paid/cloud fallback; leave work pending or enable manual correction on failure. [AI SDK compatible-provider capabilities](https://ai-sdk.dev/providers/openai-compatible-providers).

Serve private receipts through an owner- and expiry-checked route, outside `public/`. Enforce byte/type/magic-byte limits and generated opaque keys; test traversal and symlink escape in the real filesystem adapter. Strip metadata and keep bytes outside workflow results, shared caches, diagnostics and long-lived backups. Approval fixes expiry at seven days; retry, edit, undo and restore must preserve it. Physical deletion and rejected/abandoned receipt policy remain release decisions. [OWASP file upload guidance](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html), [retention requirements](./app-architecture.md#scheduled-work-reports-and-retention).

Use structured diagnostic events with operation name, opaque correlation ID, safe error code, duration and retry count. Keep raw input, cookies, credentials, receipt bytes, financial descriptions and model response bodies out by default. Review tracing SDK capture as well as application logger calls. Use bounded metric labels rather than one series per command/attachment; diagnostic logs, financial audit and metrics have different purposes and retention. OpenTelemetry describes removal/redaction and warns that hashing alone need not anonymize predictable identifiers. This is privacy guidance, not a requirement to deploy an OpenTelemetry collector. [OpenTelemetry sensitive-data handling](https://opentelemetry.io/docs/security/handling-sensitive-data/).

## Tests that help the next maintainer

Test observable behaviour through supported interfaces with hand-computed expected results. Don't compute an expected balance by calling the same posting builder that production uses. Name the case by its guarantee, such as “replaying an acknowledged transfer doesn't charge its fee twice.” Keep test setup readable, isolated and synthetic; create valid branded values through constructors instead of `as any`.

| Test layer | What belongs here | What it cannot establish alone |
|---|---|---|
| Vitest module/pure policy | Exact parsing, fee accounting, zero/positive rules, state transitions, month edges, fixed expiry, deterministic narrative fallback | SQL roles/locks, browser storage durability or a working provider |
| Adapter contracts | Shared behavioural cases against stateful fakes and actual adapters; PGlite for compatible SQL/migration feedback | Complete real-Postgres equivalence; a fake that just invokes a callback doesn't prove rollback |
| Real PostgreSQL | Migrations, constraints, RLS/runtime grants, pool-context isolation, same-key replay races, competing approval, rollback after later failure | Real filesystem behaviour or deployed proxy/session integration |
| Filesystem/HTTP/runtime | Path/symlink limits, upload bounds, private no-store reads, expiry, crash between file/SQL/send/ack, independent recovery | Installed mobile-PWA behaviour or actual restore unless exercised |
| Playwright | User-visible capture/review/replay, lost responses, auth expiry, owner switch, storage failures, old queued payload after update | Actual installed iPhone lifecycle or all host sleep/restore conditions |

The tool choices preserve the [existing proposed stack](./tech-stack.md#7-verification-required-before-implementation-can-be-called-ready). Vitest supplies the test runner; PGlite supplies embedded PostgreSQL execution where compatible. Neither tool's availability establishes the production guarantees above. [Vitest guide](https://vitest.dev/guide/), [PGlite documentation](https://pglite.dev/docs/).

Focused pure-policy tests are useful at a meaningful interface. Don't export a private helper solely to test it, and don't forbid direct SQL assertions when SQL constraints/roles are the subject. Use behavioural fakes at owned seams rather than mocks for every internal call. Keep live model/provider calls explicitly opt-in, bounded and separate from deterministic CI; evaluating actual receipt accuracy needs a permitted representative dataset and expected fields.

For Playwright, use role/label locators, retrying assertions and isolated browser contexts; avoid fixed sleeps and CSS structure assertions. Retain trace evidence for failures without putting private receipts into artifacts. Test user-visible outcomes and mock only dependencies outside the scenario's purpose. [Playwright best practices](https://playwright.dev/docs/best-practices).

Add tests in proportion to changed behaviour and risk. A financial bug requires a reproducing regression case; a wording or formatting change doesn't justify a mirrored test. Don't set an arbitrary coverage percentage as the definition of correctness. Distinct failure/concurrency cases matter more than several tests asserting the same successful path.

## Formatting, lint and delivery gates

### Minimal toolchain

Keep ESLint as already proposed. Add Prettier for authored TS/TSX, JS, JSON, CSS and Markdown, using its defaults unless a specific project need warrants an option. Let `terraform fmt` own HCL. Disable overlapping ESLint formatting rules with the appropriate Prettier integration rather than running two formatters over the same files. A formatter decides layout; ESLint catches coding defects. [Prettier comparison](https://prettier.io/docs/comparison).

Use a flat ESLint configuration with the compatible Next.js/React Hooks recommendations and `typescript-eslint` recommended type-checked rules. Add a short reviewed set: no explicit `any`, no unsafe `any` propagation, no floating/misused promises, exhaustive switches, consistent type-only imports and unused-symbol handling. Some rules already come from the preset; verify the effective configuration rather than duplicating every rule. Require an explanatory, narrow suppression for a real exception and remove it when the limitation disappears.

Type-aware linting needs project configuration and costs more than syntax-only lint. `projectService` is the documented starting point, but all checked source/test/ops files need a compatible project or deliberate override; don't enable wide fallback projects to hide configuration errors. Avoid a large collection of stylistic plugins. [typescript-eslint typed linting](https://typescript-eslint.io/getting-started/typed-linting/).

Run ESLint explicitly in CI and document that command; don't assume a production build invokes it. Next.js documents the ESLint CLI and removes `next lint` starting in version 16. Keep compatible Next/TypeScript/ESLint versions pinned during bootstrap. [Next.js ESLint configuration](https://nextjs.org/docs/app/api-reference/config/eslint).

### Proposed check contract

These script names are an implementation target, not commands available in the current docs-only repository. Select one package manager, pin supported versions and commit its lockfile; use frozen/clean installation in CI. Keep human and agent verification on the same scripts.

| Proposed script/gate | Required work |
|---|---|
| `format:check` | Formatter check for authored supported files, excluding generated/vendor/build artifacts |
| `lint` | ESLint CLI over authored application, test and tooling code; fail on actionable diagnostics |
| `typecheck` | Shared/core, browser, worker and ops checks using their actual TypeScript environments |
| `check:imports` | Resolved dependency/environment rules plus checker fixtures |
| `test` | Deterministic Vitest module and adapter-contract cases |
| `test:postgres` | Disposable real PostgreSQL with migrations and runtime-role/concurrency cases |
| `test:e2e` | Built application with Playwright critical flows and synthetic data |
| `build` / `build:worker` | Production web and standalone worker/dispatch/recovery artifacts; smoke-start them |
| Infrastructure check | `terraform fmt -check -recursive`, then initialized `terraform validate` for applicable roots/modules |

Require fast static/module checks on every code change. Run relevant database/HTTP tests for server/schema changes, browser flows for UI/offline changes, and worker/runtime cases for workflow/deployment changes; run the full implemented critical suite before release. Ensure path filters cannot silently skip a cross-cutting contract change. Local hooks are optional conveniences, while CI remains authoritative.

Review generated migrations and commit their actual metadata with custom SQL guards; apply the same migrations in tests. Prefer additive schema changes while existing clients/workflows might still replay old payloads. Don't rewrite a migration already applied to a durable environment, and don't use an unreviewed schema push as production migration policy. Drizzle supports both generated migration files and direct push; choosing reviewed migration files here is a project decision. [Drizzle migrations](https://orm.drizzle.team/docs/migrations).

Keep Terraform's shared runtime module and separate environment states. Static `validate` checks syntax/internal consistency, not live provider permissions, persistent mounts or operating cost; review the plan and test a disposable runtime. Keep secret files/state/plans out of Git. Marking a Terraform value `sensitive` redacts display without inherently removing it from state. [Terraform validate](https://developer.hashicorp.com/terraform/cli/commands/validate), [fmt](https://developer.hashicorp.com/terraform/cli/commands/fmt), [sensitive data](https://developer.hashicorp.com/terraform/language/manage-sensitive-data).

## Human and AI contributor workflow

Keep one working agreement for both. Predictable names, public contracts, tested boundaries and runnable scripts reduce the context either reader needs. No particular function size or architecture pattern guarantees that an AI agent will maintain the code correctly.

1. **Locate the current owner and decision.** Read applicable `AGENTS.md`, the relevant issue/acceptance criteria, available glossary/ADRs and the owning module contract. Use CodeGraph first when `.codegraph/` exists, as repository guidance requires; don't create an index as an incidental coding step. Read the relevant implementation and tests before changing a caller.
2. **State the behavioural change.** Identify inputs, expected result, owner authorization and any replay/rollback/expiry consequences. Resolve contradictions against the source-precedence table. Record an assumption explicitly when an open product decision affects the implementation.
3. **Make a focused vertical change.** Put the rule in its owner, wire it through existing seams and add the necessary migration/contract change with it. Avoid speculative scaffolding, unrelated renames, copied business rules and edits to generated output as if it were authored source.
4. **Verify and inspect.** Run the applicable shared scripts, review the diff and check failure cases. State what ran, what passed and what remains unverified; don't invent a passing check, benchmark or live compatibility result. Keep unresolved failures visible instead of deleting tests or weakening types/lint to pass.
5. **Leave useful evidence.** Update the public contract/runbook when behaviour changes. A change description should explain the resulting behaviour, relevant verification and material limits. Preserve unrelated local work and never include real receipts or credentials in fixtures, logs or prompts.

If these recommendations are adopted later, link the accepted standard from the contributor entry point and keep exact executable commands beside the package/ops scripts. Keep the entry point short; link domain and operating detail rather than copying this entire research note into it. AGENTS.md's primary format describes this role as a predictable source of project context and commands. This research changes no agent instruction files. [AGENTS.md format](https://agents.md/), [local domain instructions](../agents/domain.md).

### Historical examples requiring care

Do not copy the older research snippets as production-ready scaffolding:

| Earlier example | Required correction in a new implementation |
|---|---|
| `TxOrDb`/ORM types in public and internal domain functions | Use the latest core-owned persistence/atomic-work contracts; concrete Drizzle stays in adapters. |
| IDR shorthand parser using `Number` and `Math.round` | Parse exact decimal/integer values, with explicit unsupported precision/rounding behaviour. |
| Positive parser invoked for `Rp 0` opening balance | Use a constructor that permits zero; keep positive movement checks separate. |
| Tests creating identities/dates with `as any` | Build validated synthetic fixtures and keep type checks active in tests. |
| Returning a domain error from an ordinary transaction callback | Define/test rollback translation; returning a value alone doesn't abort. |
| “Never raw SQL assertions” or “never pure/internal policy tests” | Test at meaningful seams; use SQL to verify database guards and focused pure tests where useful. Don't export internals only for tests. |
| Claimed PGlite timings, universal capability/fallback guarantees or calibrated confidence | Measure the selected environment and model path; don't inherit unsupported guarantees. |

These corrections follow the current [structure proposal](./codebase-structure.md#enforcement-and-verification), [exact-value/recovery architecture](./app-architecture.md) and the primary transaction/type sources above. The older notes remain research history.

## Adoption order and open decisions

| Stage | Adopt when | Concrete result |
|---|---|---|
| Bootstrap | First implementation slice | Compatible pinned runtime/dependencies, formatter/ESLint/strict TS, documented scripts and basic CI; create only needed files. |
| Manual ledger slice | First financial write | Exact values, owner mapping, intent-based ledger interface, narrow SQL work contract, reviewed migrations; fee/replay/rollback and real RLS tests. |
| Offline slice | First persisted browser command | Versioned HTTP envelope, stable owner-scoped ID, IndexedDB acknowledgement discipline, old-command and auth/quota/lost-response cases. |
| Receipt/proposal slice | First AI capture | Private file recovery, durable dispatch, pending-only extraction, tested atomic approval, fixed expiry and bounded provider contract checks. |
| Operating release | Before relying on durable financial records | Independently supervised recovery, separate build smoke checks, installed iPhone evidence, backup restoration, expiry-lag monitoring and explicit deployment procedures. |

Add a check with the code it governs, instead of installing every proposed tool before one feature works. Exceptions need a concrete scope, reason and verification; an exception for a third-party generated declaration doesn't waive strict checking across application code.

Before these standards can settle implementation details, decide transaction abort/lock semantics, reversal/edit/opening-balance policy, unsupported IDR precision and rounding, supported offline contract versions, and receipt expiry for rejected/abandoned captures. The operating decisions also remain open: owner provisioning/recovery, permitted inference data use and budgets, seven-day physical deletion delay across copies, recovery targets and Mac/AWS roles. Multi-user admission/workspaces remain outside the single-owner MVP. A coding standard cannot choose these product contracts by implication.

## Short review checklist

- What caller knowledge does the module hide, and does a new layer/port supply a distinct capability or protect a source dependency?
- Does the change keep related rules with their owner, use consistent domain names and preserve clear control flow?
- Do types and runtime parsing express valid alternatives without `any`, unchecked casts or accidental mutable sharing?
- Are money, owner authorization, replay/conflict, rollback and expiry correct for both success and failure?
- Do imports preserve core/adapter and browser/server/worker boundaries, including transitive exports?
- Do proportionate tests verify independent expected outcomes, with real adapters where their guarantees matter?
- Did the author run the applicable checks and leave accurate contract, migration and operational evidence?

## Research limits

Reviewed the supplied guide, every Markdown research note listed in the coverage table and the repository contributor/domain guidance in full. Rechecked the original Ousterhout, Martin and Cockburn sources and selected first-party language, framework, lint, persistence and workflow documentation on 2026-11-01. Source facts and project judgments are distinguished above; linked historical examples are not adoption evidence. This revision changes only this standards research file.

This note does not establish compatible package versions, valid ESLint/tsconfig composition, migration correctness, runtime performance, receipt accuracy, deployed authorization, backup recovery or a physical deletion bound. Those require implementation and the corresponding evidence. No application tests ran because no application/test setup exists yet. No dependencies, indexes, instruction files or infrastructure were created.
