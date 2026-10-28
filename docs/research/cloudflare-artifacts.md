# Cloudflare Artifacts research

Researched 2026-10-28 for GammaDB. Compared against [docs/research/deltadb.md](deltadb.md) and [CONTEXT.md](../../CONTEXT.md).

Method, pass 1 (2026-10-28): WebFetch on first-party Cloudflare pages (docs + blog). WebFetch returns a small-model summary, not raw text, so those statements were "as relayed" and not byte-verified.

Method, pass 2 (same day): every docs page listed under "Findings from raw docs" was re-fetched as raw markdown with `curl -sS -m 30 -H "Accept: text/markdown" <page URL>` (all 22 pages plus llms.txt returned HTTP 200). Where pass 2 contradicts pass 1, the text below has been corrected in place and marked **[corrected in pass 2]**; the full list is in "Corrections from raw docs". The blog posts [8][9] were NOT re-fetched raw in pass 2, so claims sourced only to them remain "as relayed". "Stated" = on the cited page. "Inference" = mine. Citation numbers map to URLs in Sources.

## Summary

**Not similar to DeltaDB in the way that matters; it is a git-hosting primitive, not a CRDT delta layer.** Stated: Artifacts is "Versioned storage that speaks Git" [1]. Its unit of history is a git commit; the docs and launch blog say nothing about CRDTs, character-level edits, real-time sync or presence [3][8]. It overlaps with GammaDB only on the "per-thread isolation" idea (repo-per-session, fork from baseline) and on being a git-compatible store that agents can use [1][7][8].

**Possible building block, not a replacement.** Inference: it could host GammaDB's git side (Land targets, per-Thread git remotes, snapshots/archive), but the live delta log, presence, and CRDT merge would still have to be built (e.g. on Durable Objects directly). Cloudflare itself says the coordination/review/merge layer is "left to builders" [9].

## What it is (stated)

- Product: "Versioned storage that speaks Git"; create repos programmatically, import existing ones, connect via Workers, REST API or standard git clients. Requires Workers Paid [1].
- Status: private beta from 2026-04-16, "now in open beta" from 2026-10-01; no mention of GA [6]. Billing start: pricing page [4] and the 2026-10-01 changelog entry [6] both say October 14, 2026; only the blog says October 15 [9] (not re-fetched raw). **[corrected in pass 2: the docs agree on Oct 14; the blog is the outlier.]**
- Model: namespaces contain repos; repo name unique within namespace; each repo is modelled on a Durable Object (single logical instance routable from any region) with its own remote URL, refs, tokens [2][3].
- Implementation (blog): each repo backed by a Durable Object; objects stored in the DO's SQLite (rows capped at 2MB, large git objects split across rows); git server written in Zig, compiled to ~100KB WASM; R2 holds snapshots, KV tracks tokens; ~128MB DO memory so fetch/push are streamed; raw deltas are persisted rather than recomputed [8].
- Durability: synchronous replication across data centres, async copy to object storage and snapshots [2].
- API: Workers binding `env.ARTIFACTS` (`create`, `get`, `list`, `import`, `delete`; repo handle: `info`, `createToken`, `listTokens`, `revokeToken`, `fork`, `log`, `readCommit`, `readTree`, `readBlob`, `readFile`) [5]; REST API; Wrangler commands; no separate git page, git protocol is under API [1][5][6][10].
- Git protocol: clone/fetch (`upload-pack`) v1 and v2; push (`receive-pack`) v1 only ("does not support v2 receive-pack"); `filter` and `include-tag` not supported. Auth by bearer header or basic auth with token `art_v1_<40 hex>?expires=<unix>`; scopes `read` / `write` [10].
- Forks: new repo starting from an existing repo's history, diverging with own tokens and lifecycle; options `readOnly`, `defaultBranchOnly`, `description` [2][5]. Import from a **public** HTTPS git URL with optional branch/depth [17][20]. **[corrected in pass 2: pass 1 said "HTTPS git URL"; the raw docs title the route "Import a public HTTPS remote" and the guide says "Artifacts imports public HTTPS remotes". An error `REMOTE_AUTH_REQUIRED` exists (10106) [21] but no page documents passing credentials for a private source.]** Fork takes no commit/ref argument, see "Findings from raw docs".
- Events (via Queues event subscriptions): `repo.created/deleted/forked/imported`; per-repo `pushed` (ref, before, after, commits), `cloned`, `fetched`, `token.created/revoked` [11].
- ArtifactFS: FUSE driver; blobless clone, then contents hydrate asynchronously on read [15]. **[corrected in pass 2: the docs guide is at `.../guides/artifact-fs/` (hyphen; pass 1 used a wrong URL and got 404). The "does NOT sync back to remote, agents use normal git to commit and push" claim came only from the blog [8]; the raw guide page [15] says nothing about writes or sync-back, and its example mounts with a read token and only reads. So write behaviour is "blog says no sync-back; docs silent", not a docs-stated fact.]**
- Agent guidance [7]: one repo per agent/session/unit of work ("10,000 agents -> 10,000 repos"); don't use one shared repo as a queue for autonomous agents; fork each session from a trusted baseline; short-lived repo-scoped tokens, `read` by default; store harness metadata (prompts, run IDs) in git notes on `refs/notes/*` (must be pushed/fetched explicitly); shard namespaces if rate limits are hit.
- Pricing (stated) [4]: Workers Paid only. Operations: 10,000/month included, then $0.15 per 1,000. Storage: 1 GB included, then $0.50 per GB-month (average of daily peak over 30 days; summed across repos; replicas free).
- Limits (stated) [12]: 2,000 control-plane requests/10 s per namespace; 2,000 git requests/10 s per repo; 1 GB per repo for repos created on/after 2026-09-29 (10 GB for earlier); 1 TB per account (raisable); unlimited repos and namespaces. Page does not mention object-size limits.
- Jurisdiction: namespaces can be pinned to EU or US, immutable after creation [6]. Dashboard management since 2026-06-17 [6].
- Planned (blog, as of April): more metrics, SDKs (TS, Go, Python), repo/namespace search, Workers Builds API [8].

## Findings from raw docs

Source: raw markdown via `curl -H "Accept: text/markdown"`, saved outside the repo in the session scratchpad (not executed). Page "last updated" dates are from each page's own header. "Stated" unless marked inference.

**Answers up front**

- Real-time sync, CRDTs, character-level deltas, presence, collaborative editing: **absent**. The words CRDT, real-time/realtime (except the namespace name example `agents-realtime`), collaborat*, lock, merge API and emulator do not occur as features in any of the 22 pages (grep of raw text; `merge` appears only in best-practices prose [7]).
- Merge/coordination primitive: **none**. No merge, rebase, diff, compare or PR endpoint in the binding, REST API or Wrangler [5][20][22].
- Fork-from-commit: **no**. Fork copies the whole repo (optionally default branch only) [5][20].
- Push concurrency semantics, max push/pack size, object size: **not stated**. Only `MEMORY_LIMIT` (10402, "operation exceeds service memory limits") hints that large operations can fail [21].

**Per page**

1. https://developers.cloudflare.com/artifacts/concepts/how-artifacts-works/ (Apr 25, 2026) [2]. Namespace > repo; a repo is "a single logical instance that Cloudflare can route to from any region", "like Durable Objects"; each repo has own history/refs, tokens, remote URL, lifecycle. "A fork creates a new repo that starts from an existing repo's history, then diverges independently." Tokens `read` (clone, fetch, pull, indexing, review) and `write` (push and other mutations); "Your Worker or API layer decides when to mint those tokens." Durability: synchronous replication across data centres, async to object storage and snapshots.
2. https://developers.cloudflare.com/artifacts/concepts/repositories/ (Aug 13, 2026) [3]. Three identifiers: namespace, repo name, repo ID. Binding and REST are control plane; git is data plane. "Artifacts tokens are repo-scoped... does not grant access to another repository, even when both repositories live in the same namespace." "Use one repository per agent, session, user task, baseline, or fork target."
3. https://developers.cloudflare.com/artifacts/concepts/namespaces/ (Aug 13, 2026) [14]. Namespaces created implicitly by first repo or explicitly; jurisdiction fixed at creation; same namespace name must be used in the binding config and REST base URL. Name rules: start with letter/digit, then letters, digits, `.`, `_`, `-`.
4. https://developers.cloudflare.com/artifacts/concepts/best-practices/ (Apr 25, 2026) [7]. "If you have 10,000 agents, create 10,000 repos"; avoids "a hot spot for conflicts, large diffs, and accidental overwrites". "Use branches only when collaborators share the same lifecycle... Do not use one shared repo as a queue for many autonomous agents." Fork from a "trusted baseline"; example `fork(name, {defaultBranchOnly: true, readOnly: false})`. Mint least-privilege tokens, "re-issue a fresh token for each agent session" (example `createToken("read", 900)`). Git notes for prompts/model output; notes "live on separate refs. Push and fetch `refs/notes/*`". Shard namespaces (`agents-batch`, `agents-realtime`) to stay inside per-namespace rate limits.
5. https://developers.cloudflare.com/artifacts/guides/artifact-fs/ (Apr 25, 2026) [15]. See "ArtifactFS" below.
6. https://developers.cloudflare.com/artifacts/guides/authentication/ (Oct 12, 2026) [16]. Three auth paths: Workers binding (configured binding, no token in code), REST (Cloudflare API token, permissions **Artifacts > Read** / **Artifacts > Edit**), git (repo-scoped token; `read` = clone/fetch/pull, `write` adds push). "Cloudflare API tokens authenticate control-plane access. Repo-scoped Artifacts tokens authenticate Git access."
7. https://developers.cloudflare.com/artifacts/guides/event-subscriptions/ (May 21, 2026) [11]. See "Event subscriptions" below.
8. https://developers.cloudflare.com/artifacts/guides/import-repositories/ (Apr 25, 2026) [17]. Import of public HTTPS remotes via REST or binding; response carries `remote` and `token`; "An import can still be in progress after this request returns. If follow-up REST calls return `409 Conflict`, retry"; imported repos can be a "stable baseline and fork from it", or marked read-only as a template. Example uses `depth: 100`.
9. https://developers.cloudflare.com/artifacts/guides/build-and-deploy-on-push/ (Oct 26, 2026) [18]. "A `git push` to the Artifacts repo emits an `artifacts.repo.pushed` event that identifies the pushed repo, branch, and commit"; the event starts a Workflow (`@cloudflare/ci` SDK, sandboxed runners, dependency cache keyed on lockfile) that builds, checks, deploys a Worker or Workers for Platforms User Worker. Mentions Workers Builds integration as the standard path. Relevance: a working Cloudflare-documented example of push event -> Workflow -> action.
10. https://developers.cloudflare.com/artifacts/guides/data-localization/ (Aug 13, 2026) [19]. Jurisdictions `eu` and `us`, set at namespace creation (`POST /artifacts/namespaces` with `jurisdiction`), immutable; omitted = unrestricted.
11. https://developers.cloudflare.com/artifacts/api/git-protocol/ (Apr 25, 2026) [10]. Remote form `https://<ACCOUNT_ID>.artifacts.cloudflare.net/git/<namespace>/<repo>.git`. Auth: Bearer via `http.extraHeader` (recommended) or Basic with token secret as password (username ignored). Token format `art_v1_<40 hex>?expires=<unix_seconds>`. Fetch/clone: `git-upload-pack` v1 and v2 (v1 incl. shallow/deepen). Push: `git-receive-pack` v1 only; "Artifacts does not support v2 receive-pack"; `filter` and `include-tag` unsupported. Nothing on push atomicity, non-fast-forward, force push, hooks, size limits, or protected branches.
12. https://developers.cloudflare.com/artifacts/api/rest-api/ (Oct 22, 2026) [20]. Base `https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/artifacts/namespaces/$NS`. Routes: namespaces (create w/ jurisdiction, list, get, delete-if-empty); repos (create, list with `search`/`sort`/`direction`, get, delete = 202, **fork**, **import**); content (log, commit/:hash, tree/:hash, blob/:hash, file?ref&path, raw/:ref/:path); tokens (list by state, create, revoke). Fork body: `name`, `description`, `read_only`, `default_branch_only`; result includes `objects` count. Token create: `scope` default `write`, `ttl` min 60 s, max 31,536,000 s, default 86,400 s. Repo create has `default_branch`, `read_only`. No write/commit/merge routes: content routes are read-only.
13. https://developers.cloudflare.com/artifacts/api/workers-binding/ (Oct 1, 2026) [5]. Namespace: `create(name, {readOnly, description, setDefaultBranch})`, `get`, `list({limit,cursor})` (repo `status`: `ready`/`importing`/`forking`), `import`, `delete`. Repo handle (disposable, use `using`): `info`, `createToken(scope="write", ttl)`, `listTokens`, `revokeToken(tokenOrId)`, `fork(name, {description, readOnly, defaultBranchOnly})`, `log({ref="HEAD", limit<=1000, offset})` ("follows the first-parent chain"), `readCommit`, `readTree` (immediate children only), `readBlob`, `readFile({ref,path})`. `get()` "Throws if the repo does not exist or is not ready yet." Config option `remote = true` uses the remote service in local dev; needs Wrangler >= 4.145.0 for types / Blob methods with remote binding. No write-content method.
14. https://developers.cloudflare.com/artifacts/api/errors/ (May 21, 2026) [21]. Codes: ALREADY_EXISTS 10201, NOT_FOUND 10200, IMPORT_IN_PROGRESS 10302, FORK_IN_PROGRESS 10303, INVALID_INPUT 10100, INVALID_REPO_NAME 10101, INVALID_TTL 10103 (60-31,536,000 s), INVALID_URL 10104, REMOTE_AUTH_REQUIRED 10106, UPSTREAM_UNAVAILABLE 10401, MEMORY_LIMIT 10402, INTERNAL_ERROR 10400. No code for push conflict, non-fast-forward, or optimistic-concurrency failure (those would be git-protocol-level, undocumented).
15. https://developers.cloudflare.com/artifacts/api/wrangler/ (May 18, 2026) [22]. `wrangler artifacts namespaces list|get`, `repos create|list|get|delete|issue-token` (issue-token takes `--namespace`, `--scope`, `--ttl`). No fork or import command (inference: none listed on the page).
16. https://developers.cloudflare.com/artifacts/observability/metrics/ (Apr 25, 2026) [23]. GraphQL dataset `artifactsEventsAdaptiveGroups`; count and duration quantiles; dimensions repository, namespace, eventKind, eventType, time buckets; event types create, fork, push, pull, delete and errors storageLimitReached, serverError, clientError, rateLimited. 31-day query window. Per-repo granularity, so per-Thread usage is visible if repo-per-Thread (inference).
17. https://developers.cloudflare.com/artifacts/examples/sandbox-sdk-artifacts/ (Sep 29, 2026) [24]. Template `git-repo-per-sandbox`: one Artifacts repo per sandbox ID, a 1-hour write token turned into an authenticated remote passed into the sandbox as env var `ARTIFACTS_GIT_REMOTE`; sandbox then uses plain git. Warns that "your flow can race with repo creation", handle at app level.
18. https://developers.cloudflare.com/artifacts/examples/isomorphic-git/ (May 21, 2026) [25]. "The Artifacts binding creates and manages repos, but it cannot read or write files inside them — for that, you need Git." Example builds files in an in-memory FS in a Worker and `git.push`es with Basic auth. (Tension: the binding page dated Oct 1 documents read methods `readFile`/`readTree`/`readBlob` [5], so this sentence is stale or means "write"; writes still require git.)
19. https://developers.cloudflare.com/artifacts/examples/git-client/ (May 21, 2026) [26]. Control plane (REST with Cloudflare API token) mints a 1-hour read token; then plain `git clone` with `http.extraHeader`. "useful when another system owns repo discovery or access control".
20. https://developers.cloudflare.com/artifacts/platform/limits/ (Oct 21, 2026) [12]. 2,000 control-plane requests / 10 s / namespace; 2,000 git requests / 10 s / "artifact" (per repo); 1 GB/repo for repos created on/after 2026-09-29, 10 GB for earlier ones; 1 TB/account (raisable); unlimited repos and namespaces; namespace names 2-63 chars. No object/pack/push size limit listed.
21. https://developers.cloudflare.com/artifacts/platform/pricing/ (Oct 1, 2026) [4]. Workers Paid only (Free: unavailable). Billing begins 2026-10-14. Operations: first 10,000/month then $0.15 per 1,000; "such as `create`, `push`, `pull`, and `clone`". Storage: first 1 GB then $0.50/GB-mo, average of daily peak over 30 days, summed over all repos, replicas free, repos kept until deleted.
22. https://developers.cloudflare.com/artifacts/platform/changelog/ (header says Apr 21, 2026 but has entries to 2026-10-01) [6]. 2026-04-16 private beta; 2026-05-18 Wrangler commands; 2026-06-17 dashboard; 2026-08-13 data localization; 2026-09-29 1 GB cap for new repos; 2026-10-01 open beta (Workers Builds integration, events, localization, metrics). Mentions a "build the next GitHub on Cloudflare" competition (submissions to 2026-10-14). The 04-16 entry's example REST/git URLs (`artifacts.cloudflare.net/v1/api/...`) differ from the current ones in [10][20]; treat as outdated.
23. https://developers.cloudflare.com/artifacts/llms.txt [13]. Index lists exactly the pages above plus Get started (not in the requested list, not read).

**Fork-from-commit and branching (stated)**

- Fork inputs are only name, description, readOnly, defaultBranchOnly [5][20]. No ref, commit, or depth. So a fork "starts from an existing repo's history" [2] as of the source's current state; a Thread forked "at commit X" would need the baseline repo to be at X, or a fork followed by `git reset`/push of a ref in the new repo (inference; force-push behaviour undocumented).
- Forks are asynchronous: repo `status` can be `forking`; `get()` throws when not ready; REST returns 409 / `FORK_IN_PROGRESS` [5][20][21].
- Branches are ordinary git branches inside a repo. Cloudflare advises branches only for collaborators who "share the same lifecycle" [7]. There is no server-side branch API in the binding/REST (only `log` by ref and file reads by ref) [5][20].

**Merge / coordination (stated absent)**: see answers up front. Cloudflare's guidance is isolation (repo per unit of work) rather than coordination [7]. Inference: the only conflict resolution is whatever git clients do on their own clone before pushing.

**ArtifactFS (stated, [15])**: works with Artifacts remotes "and other Git repositories"; "starts with a blobless clone. It fetches commits, trees, and refs first, then mounts the working tree through FUSE"; "File contents hydrate asynchronously as tools read them. Reads only block when a requested blob is not hydrated yet, and later reads come from the local blob cache"; prioritises package manifests, dependency files, common source; deprioritises large binaries. Installed via `go install github.com/cloudflare/artifact-fs/cmd/artifact-fs@latest`; needs "a working FUSE implementation on the host"; commands `add-repo` and `daemon`. Recommended for large repos in sandboxes/VMs; "For smaller repos, a regular `git clone` is usually simpler." **Not stated on the page: how local writes are handled, whether it syncs back, commit/push flow, behaviour of concurrent writers.** Blog claim of no sync-back [8] is unverified in raw form. Inference for GammaDB: it is a read-side hydration accelerator for a Checkout's git clone, not a replacement for GammaDB's Worktree materialisation or a write path.

**Event subscriptions (stated, [11][18])**: delivered via Cloudflare Queues event subscriptions, consumed from a Worker. Account-level source `artifacts`: `repo.created`, `repo.deleted`, `repo.forked` (payload names the target namespace/repo), `repo.imported` (includes `sourceUrl`, `branch`). Repo-level source `artifacts.repo` (needs `namespace` and `repo_name`): `pushed`, `cloned`, `fetched` (empty payload), `token.created` (tokenId, scope, expiresAt), `token.revoked`. `pushed` payload: `ref`, `before`, `after`, `commits[]` (id, message, `messageTruncated`, timestamp, author, committer, parents), `totalCommitsCount`, `commitsTruncated`. Envelope has `eventSchemaVersion: 1`. Delivery guarantees, ordering and retry semantics are Queues' and are not described on the page. Inference for GammaDB: a `pushed` event on a Thread's repo is a workable trigger for "Land landed" bookkeeping, mirroring to another remote, or running CI on the Landed commit (Cloudflare's own CI guide does this [18]); because repo-level subscriptions are per repo, repo-per-Thread means one subscription per Thread unless the account-level source plus filtering is used (the account-level source lists no `pushed`, so that would not work; inference).

**Token scoping per agent/Thread (stated)**: see Comparison row. Token TTL range 60 s to 1 year; binding default scope is `write` (unsafe default; always pass `"read"` explicitly) [5]; REST default `write` and 24 h [20]. Listing shows `state` active/expired/revoked [20]. Revocation by token or ID [5][20].

**Push concurrency and protocol versions**: push v1 only; fetch v1/v2 [10]. Concurrency on simultaneous pushes to one repo: not documented. The repo is "a single logical instance" [2] (Durable-Object-like), which suggests serialised handling per repo, but that is inference and says nothing about ref-update conflict behaviour.

**Limits and pricing**: see page notes 20-21. Worked example (inference, arithmetic on stated prices): one repo-per-Thread with a push every 10 s for an hour is 360 push ops, about $0.054 at $0.15/1k beyond the free 10k/month; 1,000 such Thread-hours/month is 360k ops, about $52.50. Each client fetch/clone also counts as an operation [4], and the git rate limit is 2,000 req/10 s per repo [12].

**Self-host / emulator**: none documented. `wrangler.toml` `remote = true` "use the remote Artifacts service in local dev" [5]. Only open-source component named is ArtifactFS (github.com/cloudflare/artifact-fs) [15]. The git server itself being Zig/WASM comes from the blog only [8].

## Corrections from raw docs

What the raw pages contradicted or refined in pass 1:

1. ArtifactFS URL: `guides/artifact-fs/` (hyphen), not `guides/artifactfs/`; page now read [15].
2. ArtifactFS "does NOT sync back" and "reads block until hydrated": sync-back claim is blog-only [8], not in the docs page; read blocking is narrower than stated (only unhydrated blobs block) [15].
3. Import: public HTTPS remotes only, not "any HTTPS remote" [17][20].
4. Billing date: docs (pricing and changelog) say 2026-10-14; the Oct 15 date exists only in the blog [4][6][9].
5. Fork-from-commit open question: answered no [5][20].
6. Binding method list was incomplete: `create` also takes `setDefaultBranch`; `log` max 1000, first-parent only; `readFile` returns MIME-typed Blob; `createToken` defaults to `write` [5].
7. Token TTL (60 s to 31,536,000 s, REST default 24 h) was not recorded in pass 1 [20][21].
8. Event payload detail: `cloned`/`fetched` carry empty payloads; `pushed` has truncation flags; account-level vs repo-level sources differ [11].
9. Pass 1 said only "no merge API seen in the binding methods [5]"; confirmed absent in REST and Wrangler too [20][22].
10. Per-namespace limit wording: pass 1 "per repo" git limit is worded "per artifact" on the limits page [12]; namespace names are 2-63 chars [12].
11. Newly noticed doc inconsistency: isomorphic-git page says the binding cannot read files [25]; binding page has read methods [5].

## Comparison vs DeltaDB / GammaDB goals

DeltaDB column is from [docs/research/deltadb.md](deltadb.md) (delta.dev), not re-verified here.

| Goal | DeltaDB / GammaDB wants | Cloudflare Artifacts (stated unless marked) |
|---|---|---|
| CRDT / character-level deltas | Every edit recorded continuously as a delta; CRDT convergence | No. Unit is a git commit/pack; no CRDT or continuous edit capture mentioned in the overview, concepts or blog [2][3][8]. Per-keystroke history would have to be layered on top (inference). |
| Real-time sync and presence | Deltas sync as you type; live cursors per file | Not mentioned in any page read [2][8]. Closest feature is event subscriptions on push/fetch via Queues, which is commit-granular, not live [11]. |
| Git compatibility | Git remains system of record; Land makes a commit | Strong: standard smart-HTTP git, v1/v2 fetch, v1 push, import from any HTTPS remote [10][5]. Gaps: no v2 push, no `filter` [10]. Not a PR/issue host; blog frames it as a storage primitive [8][9]. |
| Per-agent / per-thread isolation and forks | Isolated Worktree per Thread; parallel agents don't collide | Strong fit conceptually: repo per session, fork from baseline, repo-scoped tokens [7][2][3]. Fork is whole-repo only: no commit/ref parameter in binding or REST [5][20]. Docs say "review, merge, archive, or delete work independently" and "merge back only the results you want" [7] but expose no merge, diff, compare or PR API anywhere [5][20][22]; merging is done with a git client. |
| Coordination / merge primitive | Delta log gives convergence without a merge step | None stated. Closest: best-practices advice to avoid a shared repo "hot spot for conflicts, large diffs, and accidental overwrites" and "Use branches only when collaborators share the same lifecycle" [7]. Push-race/non-fast-forward behaviour not documented in any page. |
| Token scoping | Per-Actor / per-Thread access | Per-repo only; scopes `read` and `write`; TTL 60 s to 1 year (REST default 24 h); list/revoke by id; `token.created/revoked` events [16][20][5][11]. No per-branch/ref/path scope documented (inference: none exists, since only two scopes are listed). Per-Thread scoping is achievable only by one repo per Thread [3][7]. |
| Event-driven automation | Land / mirror triggers | Stated: `pushed` event with ref, before, after, commit list (author, committer, parents, messages) via Queues event subscriptions; Cloudflare's own build/deploy guide drives a Workflow from it [11][18]. Commit-granular, not delta-granular. |
| Attribution | Every Delta attributed to one Actor; Co-Authored-By on land | Only via ordinary git author/committer fields and git notes for "agent attribution and other metadata" [8][7]. No per-edit attribution. |
| Rewind | Revert conversation + files together | Git history, diff, revert; sessions replayable via prompt and file state stored in repo [8]. Granularity is commit, not edit. |
| Hosting / pricing | Self-hosted or Cloudflare backend | Managed only, on Cloudflare; Workers Paid only; $0.15/1k ops after 10k/month, $0.50/GB-mo after 1 GB, 1 GB/repo cap for new repos, billing from 2026-10-14 [4][12][6]. Open beta, no GA statement [6]. Managed only. No self-host or emulator documented in any of the 22 raw pages; the binding's `remote = true` option "use[s] the remote Artifacts service in local dev" [5], which implies no local emulator (inference). |

## Fit as GammaDB backend (INFERENCE, not stated by Cloudflare)

Pass 2 (raw docs) confirmed the pass 1 conclusion and sharpened it: Artifacts is a per-repo git service with a control plane; it has no write-content API (writes only via git push), no merge/diff API, no fork-at-commit, no documented push-concurrency semantics, and nothing real-time.

1. **Not a drop-in for the sync server's delta store.** The `Storage` trait would need an append-only, ordered Delta log with cheap "deltas since version X" reads and live fan-out. Artifacts exposes git objects and refs, v1 push only, 2,000 git req/10 s per repo, and bills each push/pull/clone [10][12][4]. Using git commits as Deltas is too coarse and too expensive.
2. **Good fit for the git-facing side of Land.** One Artifacts repo per Thread (Cloudflare's own recommended unit [3][7]): fork from a baseline repo, mint a short-lived write token for the Landing Actor (or a read token for others), push the Landed commit, and react to the `pushed` event for mirrors, CI or bookkeeping [5][11][18]. Attribution can use commit author/trailers and git notes on `refs/notes/*` [7] (notes must be pushed explicitly).
3. **Fork limits shape the design.** Because fork cannot target a commit [5][20], a Thread based on an arbitrary git Version would need either a baseline repo kept at that commit, or push of the needed ref into a fresh repo (create + push, or import of a public remote [17]). Plan for async readiness (`forking` status, 409 retries) [5][21].
4. **ArtifactFS is optional read-side acceleration** for large repos in Checkouts [15]; GammaDB's Worktree/Checkout materialisation and write-back remain GammaDB's job. Its write/sync-back behaviour is undocumented, so do not depend on it.
5. **Cloudflare backend for GammaDB should still likely be Durable Objects + SQLite (+R2), not Artifacts.** Artifacts is described as DO-like single-instance repos [2] (and per blog built on DOs/SQLite/R2 [8]); a DO per Thread holding the CRDT log with WebSocket fan-out for Presence is the natural fit, with Artifacts beside it as the git store. I did not verify DO WebSocket details.
6. **Constraints to weigh:** open beta, no GA/SLA [6]; 1 GB/repo for new repos [12]; push v1 only [10]; Workers Paid only [4]; no self-host/emulator (local dev talks to the remote service [5]), so the self-hosted backend needs its own git store (plain git on disk) and Artifacts should be an optional adapter behind the `Storage` trait or a separate `GitRemote`-style trait.
7. **Alternative reading:** Artifacts covers the "git stays the system of record" half; GammaDB's novelty is the in-between half [deltadb.md].

## Open questions

Status key: ANSWERED (raw docs), PARTIAL, OPEN.

- Billing start date: ANSWERED. Docs say 2026-10-14 on pricing and changelog [4][6]; only the blog says Oct 15 [9].
- GA timeline and SLA: OPEN. None stated in any page [6][4].
- Hidden CRDT/real-time feature: ANSWERED (absent). None in any of the 22 pages; REST, namespaces, errors, metrics, Wrangler now read [20][14][21][23][22].
- Fork from a given commit/ref: ANSWERED, no. Whole-repo fork, optional `defaultBranchOnly` [5][20]. Blog's "from any point" is not supported by the docs.
- Max object/pack/push size: OPEN. Not in limits [12] or git-protocol [10]; only `MEMORY_LIMIT` error exists [21].
- Concurrent pushes to one repo / non-fast-forward / force-push behaviour: OPEN. Not documented anywhere; single-instance-per-repo wording [2] is the only hint.
- Cost model for high-frequency snapshotting: PARTIAL. Prices and rate limits known [4][12] with a worked example above; whether fetches by ArtifactFS and `cloned`/`fetched` count per operation is not itemised (pricing lists create/push/pull/clone "such as").
- Server-side merge: ANSWERED, none. No merge/diff/compare in binding, REST or Wrangler [5][20][22].
- Self-hosting or local emulator: ANSWERED, none documented; local dev uses `remote = true` against the real service [5].
- ArtifactFS write and sync-back behaviour: OPEN. Page silent [15]; blog says no sync-back [8].
- Private-repo import (credentials for non-public source): OPEN/PARTIAL. Docs say public HTTPS only [17][20]; `REMOTE_AUTH_REQUIRED` error exists [21].
- Per-branch or per-path token scope: PARTIAL. Only `read`/`write` documented [16][20]; none found.
- Event delivery guarantees (ordering, retries, latency) for `pushed`: OPEN on these pages; they defer to Queues event subscriptions [11] (https://developers.cloudflare.com/queues/event-subscriptions/ not read).
- Does the git server support push options, hooks, atomic multi-ref push, or protected branches? OPEN. Not mentioned [10].
- Get started pages (get-started, get-started/rest-api, get-started/workers) and Workers Builds Artifacts integration page: not read (outside the requested list).

## Pages that failed to load

- Pass 2: none. All 22 requested pages and llms.txt returned HTTP 200 with markdown.
- Not requested/not read: Get started pages, Queues event-subscriptions docs, Workers Builds Artifacts integration, ArtifactFS source repo, and the blog posts [8][9] (not re-fetched raw; still "as relayed").
- Pass 1 failure was a wrong URL (`.../guides/artifactfs/`); correct URL is `.../guides/artifact-fs/` [15].
- Third-party writeups (flaviocopes.com, lilting.ch) appeared in search and were not used.

## Sources

1. https://developers.cloudflare.com/artifacts/ (overview)
2. https://developers.cloudflare.com/artifacts/concepts/how-artifacts-works/ and https://developers.cloudflare.com/artifacts/concepts/ (concepts index)
3. https://developers.cloudflare.com/artifacts/concepts/repositories/
4. https://developers.cloudflare.com/artifacts/platform/pricing/
5. https://developers.cloudflare.com/artifacts/api/workers-binding/ (index: https://developers.cloudflare.com/artifacts/api/)
6. https://developers.cloudflare.com/artifacts/platform/changelog/
7. https://developers.cloudflare.com/artifacts/concepts/best-practices/
8. https://blog.cloudflare.com/artifacts-git-for-agents-beta/ (2026-04-16 launch post)
9. https://blog.cloudflare.com/next-git-platform-on-cloudflare
10. https://developers.cloudflare.com/artifacts/api/git-protocol/
11. https://developers.cloudflare.com/artifacts/guides/event-subscriptions/
12. https://developers.cloudflare.com/artifacts/platform/limits/
13. https://developers.cloudflare.com/artifacts/llms.txt
14. https://developers.cloudflare.com/artifacts/concepts/namespaces/
15. https://developers.cloudflare.com/artifacts/guides/artifact-fs/
16. https://developers.cloudflare.com/artifacts/guides/authentication/
17. https://developers.cloudflare.com/artifacts/guides/import-repositories/
18. https://developers.cloudflare.com/artifacts/guides/build-and-deploy-on-push/
19. https://developers.cloudflare.com/artifacts/guides/data-localization/
20. https://developers.cloudflare.com/artifacts/api/rest-api/
21. https://developers.cloudflare.com/artifacts/api/errors/
22. https://developers.cloudflare.com/artifacts/api/wrangler/
23. https://developers.cloudflare.com/artifacts/observability/metrics/
24. https://developers.cloudflare.com/artifacts/examples/sandbox-sdk-artifacts/
25. https://developers.cloudflare.com/artifacts/examples/isomorphic-git/
26. https://developers.cloudflare.com/artifacts/examples/git-client/
