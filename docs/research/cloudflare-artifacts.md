# Cloudflare Artifacts research

Researched 2026-10-28 for GammaDB. Compared against [docs/research/deltadb.md](deltadb.md) and [CONTEXT.md](../../CONTEXT.md).

Method: WebFetch on first-party Cloudflare pages (docs + blog). WebFetch returns a small-model summary, not raw text, so statements are "as relayed" and not byte-verified. "Stated" = on the cited page. "Inference" = mine.

## Summary

**Not similar to DeltaDB in the way that matters; it is a git-hosting primitive, not a CRDT delta layer.** Stated: Artifacts is "Versioned storage that speaks Git" [1]. Its unit of history is a git commit; the docs and launch blog say nothing about CRDTs, character-level edits, real-time sync or presence [3][8]. It overlaps with GammaDB only on the "per-thread isolation" idea (repo-per-session, fork from baseline) and on being a git-compatible store that agents can use [1][7][8].

**Possible building block, not a replacement.** Inference: it could host GammaDB's git side (Land targets, per-Thread git remotes, snapshots/archive), but the live delta log, presence, and CRDT merge would still have to be built (e.g. on Durable Objects directly). Cloudflare itself says the coordination/review/merge layer is "left to builders" [9].

## What it is (stated)

- Product: "Versioned storage that speaks Git"; create repos programmatically, import existing ones, connect via Workers, REST API or standard git clients. Requires Workers Paid [1].
- Status: private beta from 2026-04-16, "now in open beta" from 2026-10-01; no mention of GA [6]. Blog says billing starts October 15, 2026 [9]; the pricing doc says October 14, 2026 [4] (discrepancy; see Open questions).
- Model: namespaces contain repos; repo name unique within namespace; each repo is modelled on a Durable Object (single logical instance routable from any region) with its own remote URL, refs, tokens [2][3].
- Implementation (blog): each repo backed by a Durable Object; objects stored in the DO's SQLite (rows capped at 2MB, large git objects split across rows); git server written in Zig, compiled to ~100KB WASM; R2 holds snapshots, KV tracks tokens; ~128MB DO memory so fetch/push are streamed; raw deltas are persisted rather than recomputed [8].
- Durability: synchronous replication across data centres, async copy to object storage and snapshots [2].
- API: Workers binding `env.ARTIFACTS` (`create`, `get`, `list`, `import`, `delete`; repo handle: `info`, `createToken`, `listTokens`, `revokeToken`, `fork`, `log`, `readCommit`, `readTree`, `readBlob`, `readFile`) [5]; REST API; Wrangler commands; no separate git page, git protocol is under API [1][5][6][10].
- Git protocol: clone/fetch (`upload-pack`) v1 and v2; push (`receive-pack`) v1 only ("does not support v2 receive-pack"); `filter` and `include-tag` not supported. Auth by bearer header or basic auth with token `art_v1_<40 hex>?expires=<unix>`; scopes `read` / `write` [10].
- Forks: new repo starting from an existing repo's history, diverging with own tokens and lifecycle; options `readOnly`, `defaultBranchOnly`, `description` [2][5]. Import from HTTPS git URL with optional branch/depth [5].
- Events (via Queues event subscriptions): `repo.created/deleted/forked/imported`; per-repo `pushed` (ref, before, after, commits), `cloned`, `fetched`, `token.created/revoked` [11].
- ArtifactFS: open-source driver; blobless clone then background hydration; reads block until file hydrated; does NOT sync back to remote, agents use normal git to commit and push [8]. (The docs guide URL `.../guides/artifactfs/` returned 404 when fetched; see below.)
- Agent guidance [7]: one repo per agent/session/unit of work ("10,000 agents -> 10,000 repos"); don't use one shared repo as a queue for autonomous agents; fork each session from a trusted baseline; short-lived repo-scoped tokens, `read` by default; store harness metadata (prompts, run IDs) in git notes on `refs/notes/*` (must be pushed/fetched explicitly); shard namespaces if rate limits are hit.
- Pricing (stated) [4]: Workers Paid only. Operations: 10,000/month included, then $0.15 per 1,000. Storage: 1 GB included, then $0.50 per GB-month (average of daily peak over 30 days; summed across repos; replicas free).
- Limits (stated) [12]: 2,000 control-plane requests/10 s per namespace; 2,000 git requests/10 s per repo; 1 GB per repo for repos created on/after 2026-09-29 (10 GB for earlier); 1 TB per account (raisable); unlimited repos and namespaces. Page does not mention object-size limits.
- Jurisdiction: namespaces can be pinned to EU or US, immutable after creation [6]. Dashboard management since 2026-06-17 [6].
- Planned (blog, as of April): more metrics, SDKs (TS, Go, Python), repo/namespace search, Workers Builds API [8].

## Comparison vs DeltaDB / GammaDB goals

DeltaDB column is from [docs/research/deltadb.md](deltadb.md) (delta.dev), not re-verified here.

| Goal | DeltaDB / GammaDB wants | Cloudflare Artifacts (stated unless marked) |
|---|---|---|
| CRDT / character-level deltas | Every edit recorded continuously as a delta; CRDT convergence | No. Unit is a git commit/pack; no CRDT or continuous edit capture mentioned in the overview, concepts or blog [2][3][8]. Per-keystroke history would have to be layered on top (inference). |
| Real-time sync and presence | Deltas sync as you type; live cursors per file | Not mentioned in any page read [2][8]. Closest feature is event subscriptions on push/fetch via Queues, which is commit-granular, not live [11]. |
| Git compatibility | Git remains system of record; Land makes a commit | Strong: standard smart-HTTP git, v1/v2 fetch, v1 push, import from any HTTPS remote [10][5]. Gaps: no v2 push, no `filter` [10]. Not a PR/issue host; blog frames it as a storage primitive [8][9]. |
| Per-agent / per-thread isolation and forks | Isolated Worktree per Thread; parallel agents don't collide | Strong fit conceptually: repo per session, fork from baseline, repo-scoped tokens, compare/merge afterwards [7][2][9]. Isolation is at repo level via git, merge is ordinary git merge (merging not discussed beyond standard git [8]). |
| Attribution | Every Delta attributed to one Actor; Co-Authored-By on land | Only via ordinary git author/committer fields and git notes for "agent attribution and other metadata" [8][7]. No per-edit attribution. |
| Rewind | Revert conversation + files together | Git history, diff, revert; sessions replayable via prompt and file state stored in repo [8]. Granularity is commit, not edit. |
| Hosting / pricing | Self-hosted or Cloudflare backend | Managed only, on Cloudflare; Workers Paid; $0.15/1k ops, $0.50/GB-mo, 1 GB/repo cap for new repos [4][12]. Open beta, no GA statement [6]. Not self-hostable (no self-host statement found; inference). |

## Fit as GammaDB backend (INFERENCE, not stated by Cloudflare)

1. **Not a drop-in for the sync server's delta store.** The `Storage` trait would need an append-only, ordered Delta log with cheap reads of "deltas since version X" and subscriptions for live fan-out. Artifacts exposes git objects and refs, not that. Using git commits as Deltas would be far too coarse and expensive (each op billed; 2,000 git req/10 s per repo [12]).
2. **Good fit for the git-facing side of Land.** A per-Thread (or per-Checkout) Artifacts repo as the git remote/clone source: fork from the project baseline, give an agent a short-lived write token, receive the Landed commit via the `pushed` event [5][7][11]. Also useful as a cheap mirror/archive of landed commits and of Worktree snapshots (e.g. periodic snapshot commits on a `gammadb/thread-<id>` ref, with git notes for Actor/prompt metadata [7]).
3. **Cloudflare backend for GammaDB should likely be Durable Objects + SQLite (+R2), not Artifacts.** Artifacts itself is built that way [8], which is evidence the primitives suit a per-repo/per-Thread single-writer room: one DO per Thread holding the CRDT log, WebSocket fan-out for Presence, SQLite for Deltas, R2 for snapshots. Artifacts could sit beside that as the git store. This is a design suggestion; I did not verify DO WebSocket details in this research.
4. **Constraints to weigh:** open beta (not GA); 1 GB/repo for new repos; push is v1 only; not self-hostable, so the self-hosted GammaDB backend needs its own git store anyway (plain git on disk) and Artifacts would be an optional adapter behind a trait.
5. **Alternative reading:** Artifacts is a competitor/complement to git hosting for agents, i.e. it covers the "git stays the system of record" half, while GammaDB's novelty is the "everything in between" half [deltadb.md].

## Open questions

- Billing start date conflict: docs pricing page says October 14, 2026 [4]; blog says October 15, 2026 [9]. Which is current?
- GA timeline and SLA: none stated [6][9].
- Any hidden CRDT/real-time feature? None found in pages read; Workers binding sub-pages for REST API, namespaces, errors, metrics were not read in detail.
- Does `fork` support forking from a given commit/ref (blog says "from any point" for sessions) or only whole-repo [2][8]? Binding options listed only `description`, `readOnly`, `defaultBranchOnly` [5].
- Max object/pack size and push size, concurrency semantics for simultaneous pushes to one repo (non-fast-forward handling) are not stated.
- Cost model for high-frequency snapshotting: ops are billed per push/pull/clone [4]; need to model a Thread landing or checkpointing every N seconds.
- Can GitHub-style merge be done server-side in Artifacts (no merge API seen in the binding methods [5])?
- Self-hosting or a local emulator for Artifacts (needed for GammaDB's self-hosted path): none found.

## Pages that failed to load

- https://developers.cloudflare.com/artifacts/guides/artifactfs/ returned 404 (ArtifactFS facts above come from the launch blog [8] and the llms.txt listing [13]).
- Not fetched: REST API, Namespaces, Errors, Wrangler, Authentication, Metrics, Import, Data localization, Workers Builds pages; Cloudflare changelog post 2026-04-16 and Agents Week session were seen only as search results.
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
