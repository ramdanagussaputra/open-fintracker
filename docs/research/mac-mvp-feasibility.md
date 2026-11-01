# Mac MVP feasibility: constraints and verification gates

Researched 2026-11-01 (Asia/Jakarta) for [Verify the proposed stack against Mac and mobile constraints](https://github.com/ramdanagussaputra/open-fintracker/issues/2), part of [Build the single-owner financial tracker on Mac](https://github.com/ramdanagussaputra/open-fintracker/issues/1).

## Finding and evidence boundary

The proposal has documented building blocks for a Mac-hosted application, but no source establishes the complete combination as working. The main constraints are a shared supported Node release, two reachable Inngest addresses, recoverable orchestration stores, a Google-accepted phone callback origin, and browser storage that can lose unsynced captures. Installed 9router metadata also differs from the historical proposed version.

This investigation preserves the [seven-note planning snapshot at `0f1f9df`](https://github.com/ramdanagussaputra/open-fintracker/tree/0f1f9df/docs/research), applying the overview's precedence: `tech-stack.md` for infrastructure, `codebase-structure.md` for dependencies and ownership, and `coding-standards.md` for proposed checks. It doesn't adopt replacement packages, operating targets or provider/privacy policies. Diagrams aren't part of this Markdown snapshot.

**Documented** below means official documentation, source or published metadata supports the stated capability. **Observed** means a local read or registry response on this date. **Inference** identifies the project conclusion. All runtime behaviour remains **untested**: no installation, build, lockfile resolution, provider request, credential inspection, signup or infrastructure provisioning occurred.

## Node, Next.js, React and package metadata

Observed publisher metadata from `registry.npmjs.org/<package>/latest` returned these versions and constraints. Version-specific links preserve the evidence rather than relying on a moving `latest` tag.

| Published package | Declared constraint relevant here |
|---|---|
| [`next` 16.4.3](https://registry.npmjs.org/next/16.4.3) | Node `>=20.9.0`; React and React DOM `^18.2.0` or the listed React 19 ranges, including `^19.0.0`. |
| [`react` 19.3.1](https://registry.npmjs.org/react/19.3.1), [`react-dom` 19.3.1](https://registry.npmjs.org/react-dom/19.3.1) | React DOM requires React `^19.3.1`. Matching these releases satisfies that declared peer range. |
| [`inngest` 4.26.2](https://registry.npmjs.org/inngest/4.26.2) | Node `>=20`; TypeScript peer `>=5.8.0`. Connect separately requires Node `>=22.4.0` and a long-running process. [Connect requirements](https://www.inngest.com/docs/durable-execution/deploying-functions/connect). |
| [`ai` 7.0.159](https://registry.npmjs.org/ai/7.0.159), [`@ai-sdk/openai-compatible` 3.0.82](https://registry.npmjs.org/@ai-sdk%2Fopenai-compatible/3.0.82) | Both require Node `>=22` and Zod `^3.25.76 || ^4.1.8`. |
| [`vitest` 5.0.5](https://registry.npmjs.org/vitest/5.0.5) | Node `^22.12.0 || ^24.0.0 || >=26.0.0`; Vite peer `^6.4.0 || ^7.0.0 || ^8.0.0`. |
| [`better-auth` 1.7.15](https://registry.npmjs.org/better-auth/1.7.15) | Declares Next 14/15/16 and React 18/19 ranges; Drizzle ORM `^0.45.2 || >=1.0.0-rc.1 <2.0.0`, Kit `>=0.31.4 || >=1.0.0-beta.1`, and pg `^8.0.0`. |
| [`drizzle-orm` 0.45.4](https://registry.npmjs.org/drizzle-orm/0.45.4), [`drizzle-kit` 0.31.11](https://registry.npmjs.org/drizzle-kit/0.31.11), [`pg` 8.23.1](https://registry.npmjs.org/pg/8.23.1) | These versions fit the preceding declared ranges; ORM accepts pg `>=8`, while pg declares Node `>=16.0.0`. |

**Inference:** Node 24 is one candidate common major for the listed constraints; Node 22 must be at least 22.12 for this Vitest release. Next's minimum alone is insufficient. Node's release table currently lists 22, 24 and 26 as LTS and 20 as EOL. Select a supported patch during implementation, resolve the full dependency tree and commit one lockfile; this table doesn't prove transitive, native-module, browser or build compatibility. [Node release policy and status](https://nodejs.org/en/about/previous-releases).

## Self-hosted Inngest and recovery

**Documented:** `inngest start` serves API/dashboard on `8288` and Connect on `8289`. Configure self-host clients with `INNGEST_DEV=0`, `INNGEST_BASE_URL`, event key and signing key. Default persistence uses SQLite at `./.inngest/main.db` plus an in-memory Redis queue/state store whose snapshots go to SQLite periodically and before shutdown. External Redis uses `--redis-uri`; SQLite location uses `--sqlite-dir`. Thus an external persisted Redis is a proposal, not a default guarantee. [Self-hosting configuration](https://www.inngest.com/docs/self-hosting).

Connect obtains connection information over HTTP before establishing its persistent WebSocket. An explicit `gatewayUrl`, for example `ws://inngest:8289/v0/connect`, can override an unsuitable advertised address. Bundled workers must externalize `inngest` and its subpaths and include the installed package at runtime so its worker-thread runner remains available. [Connect topology and bundling](https://www.inngest.com/docs/durable-execution/deploying-functions/connect).

**Inference:** same-network containers can use service DNS for both addresses; `localhost` inside the worker names that worker's container. Both ports must be reachable from the worker, without requiring either on the public internet. Prove the exact pinned server/SDK handshake rather than copying the example hostname.

Redis distinguishes RDB snapshots, AOF and no persistence; RDB can lose writes since its previous snapshot, while AOF durability depends on fsync policy. A mounted `/data` directory alone doesn't specify the recovery loss window. [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/). **Untested:** abrupt termination, Mac sleep, delayed crons, event acknowledgement and joint restoration of SQLite/Redis/application state. Keep PostgreSQL dispatch/reconciliation authoritative as proposed; agree recovery targets before choosing persistence settings.

## Google/Better Auth over private HTTPS

Google requires an exact registered redirect URI, HTTPS except for localhost, a non-IP host except for loopback, and a valid public-suffix TLD. `.local`, bare machine names and LAN/Tailscale raw IPs aren't substitutes for an accepted hostname. Better Auth's default Google callback is `/api/auth/callback/google`; explicitly configure `baseURL` to produce the registered origin and retain its trusted-origin checks. [Google web flow and URI validation](https://developers.google.com/identity/protocols/oauth2/web-server), [Better Auth Google setup](https://better-auth.com/docs/authentication/google), [origin protection](https://better-auth.com/docs/reference/security).

**Inference:** a permitted HTTPS domain resolving privately can fit the browser-redirect flow if the phone can reach it and the Mac can exchange the code with Google. This isn't proof that the actual tailnet hostname will pass Google's Console/domain checks. `localhost` on a phone refers to the phone, so desktop localhost success doesn't validate mobile sign-in. Google requires domains the developer owns or has permission to use; its policy distinguishes personal use from production-app requirements. [Google OAuth policies](https://developers.google.com/identity/protocols/oauth2/policies).

Tailscale documents HTTPS certificates for full `machine.tailnet.ts.net` names, not bare hostnames; certificate transparency publishes those names while network access stays private. **Human setup gate:** select the origin, validate it in Google Console, provision the OAuth client and owner mapping, then test login/cookies/logout from both installed phone PWAs. No account or hostname was inspected here. [Tailscale HTTPS](https://tailscale.com/docs/how-to/set-up-https-certificates).

## iPhone and Android offline limits

**Documented:** Next supports a manifest and service-worker implementation; installation by itself doesn't establish offline behaviour. [Next PWA guide](https://nextjs.org/docs/app/guides/progressive-web-apps). WebKit uses best-effort storage by default, can evict an origin, and can reject writes with `QuotaExceededError`; standalone Home Screen apps share browser-app quota rules. `persist()` requests receive heuristic decisions, including Home Screen use. [WebKit storage policy](https://webkit.org/blog/14403/updates-to-storage-policy/). Chrome's persistent storage can protect against automatic eviction, but users can still clear it. [Chrome storage guidance](https://web.dev/articles/persistent-storage).

Safari's Background Sync request remains open in WebKit's tracker. Chrome documents browser-managed retry timing; Workbox's unsupported-browser fallback only runs when the service worker starts. Neither provides an agreed replay deadline. [WebKit request](https://bugs.webkit.org/show_bug.cgi?id=201866), [Chrome background-sync behaviour](https://developer.chrome.com/docs/workbox/modules/workbox-background-sync).

**Inference:** retain startup/foreground/manual retry paths on every supported phone, using Background Sync only when available. A successful IndexedDB transaction can justify “saved on this device”; only a server acknowledgement can justify “posted.” Test offline cold launch after an online visit, app kill/reopen, quota/write failures, storage clearing, schema upgrades, expired sessions, duplicate replay and VPN loss on real iPhone Safari/Home Screen and Android Chrome/installed PWA. These tests cannot promise survival after browser data deletion or device loss. Owner choices still govern unsynced-data handling and supported devices.

## Terraform and Docker on Mac

The `kreuzwerker/docker` provider manages containers/images through an existing Docker API; it doesn't install or start the Mac's container engine. Docker Desktop requires supported macOS and at least 4 GB RAM, offers separate Apple-silicon/Intel builds, and has license conditions. These prerequisites don't establish enough resources for the entire proposed stack. [Provider documentation](https://raw.githubusercontent.com/kreuzwerker/terraform-provider-docker/master/docs/index.md), [Docker Mac requirements](https://docs.docker.com/desktop/setup/install/mac-install/).

Docker documents the per-user `~/.docker/run/docker.sock` and optional `/var/run/docker.sock` symlink; the CLI's selected context can hide a socket mismatch in other clients. **Gate:** with the owner's selected engine running, verify the actual provider socket, image architecture, mounts/permissions and restart after login/reboot. Then verify Terraform plan/apply behaviour and application health under Mac sleep/restart. This research didn't query the daemon, inspect Docker credentials, select an engine or change startup settings. [Docker Mac permissions/socket](https://docs.docker.com/desktop/setup/install/mac-permission-requirements/).

## Installed 9router contract and provider limits

**Observed local metadata:** `command -v 9router` resolves through NVS to the installed package; `/Users/massivemusic/.nvs/node/24.18.0/arm64/lib/node_modules/9router/package.json` reports **0.5.95**, Node `>=18.0.0`. Its packaged README documents `http://localhost:20128/v1`. The registry's [0.5.95 record](https://registry.npmjs.org/9router/0.5.95) identifies gitHead `a99cf57239ff778b61e434c2786009d5ed1c412c`; [latest observed 0.5.112](https://registry.npmjs.org/9router/0.5.112) differs. This doesn't identify the running process, configured providers or binary/source equivalence.

At that published gitHead, official source defines POST chat completions and GET model discovery, including Combos and provider catalogs; model listing can itself contact upstream resolvers, so this research didn't call it. [Chat route](https://github.com/decolua/9router/blob/a99cf57239ff778b61e434c2786009d5ed1c412c/src/app/api/v1/chat/completions/route.js), [models route](https://github.com/decolua/9router/blob/a99cf57239ff778b61e434c2786009d5ed1c412c/src/app/api/v1/models/route.js). The image translator accepts base64 data URIs and blocks remote image fetching from private/CGNAT addresses. **Inference:** send bounded receipt bytes as inline image content instead of a private receipt URL, then verify the selected provider translator with synthetic images. [Image source](https://github.com/decolua/9router/blob/a99cf57239ff778b61e434c2786009d5ed1c412c/open-sse/translator/concerns/image.js).

Its README documents `X-9Router-Token-Saver: off`, prompt-altering token savers and full request/response debug logging. AI SDK supports a custom base URL, bearer credentials and headers, but describes multimodal support as provider-dependent and structured output as an opt-in capability. [Pinned router README](https://github.com/decolua/9router/blob/a99cf57239ff778b61e434c2786009d5ed1c412c/README.md), [AI SDK compatible provider](https://ai-sdk.dev/providers/openai-compatible-providers).

Router fallback claims don't prove an upstream's authorization, price or privacy contract. For example, Gemini limits vary by model/project; its unpaid-service terms allow product improvement and human review of content and prohibit submitting sensitive/confidential/personal information there. [Gemini limits](https://ai.google.dev/gemini-api/docs/rate-limits), [Gemini terms](https://ai.google.dev/gemini-api/terms). **Gate:** owner approves each destination, data handling and fallback; then verify exact task/model aliases, vision/JSON/error behaviour, logging and absence of paid fallback. No provider selection or inference occurred.

## Recurrence compatibility

Published [`rrule-temporal` 2.2.8 metadata](https://registry.npmjs.org/rrule-temporal/2.2.8) declares `temporal-spec ^1.0.0`, no Temporal peer dependency and no Node engine; [`temporal-polyfill` 1.0.6](https://registry.npmjs.org/temporal-polyfill/1.0.6) uses `temporal-spec 1.0.1`. The tagged README documents its bundled fallback, normalization of Temporal inputs and optional explicit Temporal namespace for output types. Different implementations needn't pass each other's `instanceof` tests. [Tagged package](https://github.com/ggaabe/rrule-temporal/blob/v2.2.8/package.json), [tagged interoperability guide](https://github.com/ggaabe/rrule-temporal/blob/v2.2.8/README.md#temporal-implementations-and-interoperability).

**Inference:** the historical pair has no declared dependency-range conflict; that doesn't validate runtime/type compatibility. Inngest also declares its own `temporal-polyfill ^0.2.5`; keep explicit imports and review the resolved tree rather than assuming one shared global. **Gate:** typecheck/build the pinned pair and exercise Asia/Jakarta boundaries, leap/short months, last-day versus skipped dates, DTSTART/UNTIL inclusivity and repeated occurrence materialization. Owner decisions still govern scheduling and automatic-posting semantics.

## What can proceed

Research has no remaining information blocker. Implementation may select and verify a pinned dependency set and prepare a synthetic local runtime; release remains gated by the experiments above. Human decisions still own the HTTPS identity setup, permitted AI destinations, offline loss handling and operating/recovery/deletion targets. Neither this report nor closing its research ticket marks those decisions or runtime gates complete.
