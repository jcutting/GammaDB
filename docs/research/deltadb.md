# DeltaDB research

Researched 2026-10-28 for the GammaDB project (own version of "DeltaDB" for git + coding agents).

## Method and reliability caveats

- Three primary pages were fetched with WebFetch: [A] https://delta.dev/deltadb, [B] https://delta.dev/docs/concepts/delta-and-git, [C] https://delta.dev/docs/concepts/worktrees. All three loaded.
- Two further delta.dev pages were fetched to resolve claims the primaries pointed at: [D] https://delta.dev/docs/concepts/core-concepts and [E] https://delta.dev/docs/agents/review-and-sync.
- WebFetch passes each page through a small summarising model, so I never saw the raw page text. Quotes below are short phrases as returned by the tool; I could not byte-verify them. Treat them as high-fidelity but not verbatim-certified. For anything load-bearing, re-read the page.
- "Docs state" = stated on the cited page. "Inference" = mine, labelled as such. No secondary write-ups were used. Blog/talk links from [A] (e.g. https://delta.dev/blog/future-of-version-control, https://zed.dev/blog/introducing-deltadb, the YouTube talk) were NOT read, and are the likeliest place for the technical detail [A] only gestures at.
- Naming: the product is "Delta" (an app/CLI for agent threads); "DeltaDB" is its database/version-control layer. The docs [B] describe DeltaDB as "the shared database behind a Delta thread". Much of the worktree/sync material describes the Delta app, not DeltaDB in isolation.

## Summary

- Docs state DeltaDB is a version-control layer that records edits plus the conversation that produced them: "Git keeps the commits; DeltaDB keeps everything in between" [A].
- Every edit by a person or agent becomes a "delta" (stable identity, author, the version it builds on), created continuously with nothing to stage or commit [A][B]. Built on CRDTs so concurrent edits converge on every replica [A].
- It does not replace git: a git remote is required, and you commit/open PRs as usual [A]. Work comes back to git via push to a `local` remote, push to the source remote, or in place [B][E].
- A "Delta worktree" is per-thread, per-project history recorded in DeltaDB, explicitly not a git worktree; each thread gets an isolated one by default, which is how parallel agents avoid colliding [C][D].
- The pages say little about conflict resolution, storage format, wire protocol, API or guarantees beyond convergence; many open questions remain (see below).

## Concepts & glossary

All entries are docs-stated unless marked Inference.

| Term | Meaning | Source |
|---|---|---|
| DeltaDB | "The shared database behind a Delta thread"; records messages, comments, worktree changes, file edits. Described as a version control system recording edits "along with the conversation that produced them". | [A][B] |
| Delta (the unit) | A recorded change noting who made it and where it sits in the thread's history; "Unlike a git commit, a delta is created continuously as work happens." Carries stable identity, author, and the version it builds on. | [A][B] |
| Thread | A conversation with an agent; persists; can include projects; "each thread works in its own isolated copy by default". Can be reverted (conversation and files together) and shared with collaborators. | [D][A] |
| Project | The opened folder; must be "the top-level folder of a Git repository with a configured remote"; if several remotes, one must be named `origin`. | [D][B][C] |
| Delta worktree | "a copy of the project's files whose history is recorded in DeltaDB"; one per thread per project; term "refers to this shared project history". Not a git worktree. Default setup called "Isolated Workspace". | [D] |
| Checkout | On desktop, the Delta worktree's files materialised on disk for terminal/build/editor. In browser, the agent uses the Delta worktree directly. Each thread participant gets their own checkout, kept synced. Delta also calls these "managed checkouts". | [D][C][A] |
| Replica / client | Each client holds a replica; deltas sync "as you type". | [A] |
| Source remote | The repo's real remote (often `origin`). | [B] |
| `local` remote | Remote in a managed checkout that points back to the user's own repository. | [B][C] |
| Fragment index | "B-tree fragment index maps positions to the deltas that created them" (internal design). | [A] |
| Structural sharing | Versions share structure "so storage grows with edit size". | [A] |
| Land subthread | Forked thread that runs the project's Land skill to merge/publish. | [E] |
| Verdict / Pull Changes | Review ends in Approve/Request Changes; Pull Changes applies the snapshot saved with the verdict. | [E] |

What is recorded: messages, comments, file tree changes, and edits to files Git tracks or doesn't ignore [A]. Ignored files (build output, `.env`) "stay on their original machine" and are not recorded or synced [A][B]. Warning: `.gitignore` does not protect files git already tracks [B].

## DeltaDB and git

Docs state:

- Division of labour: git holds commits, branches, remotes; DeltaDB records the work between commits [A]. "Does DeltaDB replace Git? No. You still need a Git remote" [A].
- Import on open: tracked files, uncommitted changes to tracked files, and untracked non-ignored files; line endings preserved and `core.autocrlf`, `core.eol`, `.gitattributes` honoured [B].
- Delta records edits without creating git commits. If the agent does commit, the commit lands in the managed checkout, not your project folder [E]. "Committing is not the step that brings work to you; pushing to `local` or your source remote is." [E]
- Each managed checkout is a separate folder "with its own Git repository" [E]; it has two remotes: the source remote and `local` [B].
- Getting work out, four routes: in place in your folder, push to `local`, push to source remote, or work directly in a managed checkout [B]. Concretely: `git push local <branch>` then `git switch <branch>`; if the checkout is clean with that branch checked out the push updates the working tree, and "Git rejects a push that would overwrite uncommitted edits to tracked files" [E]. Or have the agent push to origin and open a PR, then `git fetch origin` and `git switch --track origin/<branch>` [E].
- Remote status row reports Up to date / Ahead / Behind / Diverged; checking fetches for comparison only, "it does not merge" [E].
- New isolated threads fetch the remote and start from its default branch (or a configured "default branch for Delta worktrees"); fetching doesn't change the primary checkout [C].
- Diff base: "Delta records the diff's merge base and shares it with collaborators" [E].
- Merging worktrees: agent can merge selected worktrees from related ("same thread family") threads; file changes only, no conversation copied, sources unchanged; partial failure may leave earlier merges applied [E].
- Landing: Land Changes runs a user-defined Land skill; "Delta imposes no Git workflow of its own"; not available in browser; landing does not merge files into the parent thread [E].
- Jujutsu: early, incomplete support for colocated `jj` repos; Delta runs `jj git init --colocate` in managed checkouts [B].

Mapping to git concepts (largely Inference, since docs give no formal mapping):

- Commit: no delta-to-commit mapping is documented. Deltas are finer than commits; commits are made in the managed checkout's git repo by the agent/user.
- Branch/ref: the thread's isolated worktree starts from a remote default-branch tip [C]; branches seem to be ordinary git branches created in the managed checkout and transported by push. DeltaDB itself is not described as having branches; threads/forks play that role (forking a thread to try other approaches [A]) (Inference).
- Merge/sync: two layers. Inside DeltaDB, CRDT convergence handles concurrent edits [A]. Between DeltaDB and git, plain git push/fetch/switch plus agent-driven "merge worktrees" and Land skills [E]. How git-level conflicts are resolved is not documented [E].

## Worktrees

Docs state:

- Delta does not use git worktrees: "Delta does not use git worktrees for its managed checkouts." [C]. A Delta worktree is DeltaDB history, mounted as a checkout [D][A]. Delta "mounts replicated files into a temporary checkout" [A].
- One Delta worktree per thread per project; one project can be open in many threads [D]. Isolated by default; threads can instead be pointed at the existing local checkout, but then "threads sharing a checkout aren't isolated, so concurrent edits can conflict" [C].
- Lifecycle: managed checkout removed when its thread is archived, recreated on reopen [C]; "no directories or branches to clean up" [A].
- Location: `.delta` directory inside your clone if you have one, else a fully managed directory [C].
- Setup hooks: `.agents/prepare` (or `.delta/prepare`) runs on checkout creation, should be fast (no downloads/builds); `.agents/linked` (or `.delta/linked`) manifest symlinks git-ignored files such as `.env` to a shared location [C].
- Moved/deleted project folder: Delta warns and allows relocation only if a git remote confirms same repo; isolated threads keep working [C].
- Changes in the Delta worktree stay separate from your checked-out files until you bring them in [D]. Desktop checkout and worktree are kept "in sync" [D]. Participants of a shared thread each get a checkout, synced in real time [C].
- Multi-agent support (docs): "Threads run in parallel with isolated copies by default" [D]; subagents/forking exist but their pages (https://delta.dev/docs/agents/subagents) were not read.

Inference on concurrency: because each agent has its own CRDT-backed worktree plus its own git repo in a separate directory, agents cannot clobber each other's files; integration happens at an explicit step (push/pull, merge-worktrees, Land), where ordinary git conflict handling would apply. This is my reading, not stated.

## Implications for a clone aimed at git + coding agents

All Inference, derived from the above.

1. Keep git as the system of record for commits/branches/remotes; the new layer should capture the sub-commit stream (edits, messages, comments) and never require commits.
2. Core primitive: an append-only log of operations/deltas with author identity (human vs agent), parent version(s), and a thread/conversation attachment. A CRDT text model is what [A] describes (diamond-types-like); a simpler first version could use a single-writer-per-worktree op log with snapshots and defer multi-writer CRDT merging, but that loses the "concurrent edits converge" property.
3. Per-agent isolated worktree = per-agent materialised checkout directory with its own git repo (not `git worktree`), seeded from remote default branch, with a `local`-style remote back to the user repo. Cheap to create and delete; ignored files excluded, with a link manifest for secrets/env and a prepare hook.
4. Hand-off to git should be explicit and boring: push a branch to a `local` remote; leave merge strategy to a user-defined "land" step.
5. Rewind/fork: revert conversation and file state to a message boundary, requires snapshots keyed by message/turn. Structural sharing keeps storage proportional to edit size [A].
6. Record the diff merge base for reviews so all viewers compare against the same base [E].
7. Scope decisions for v1: local-first single machine (no browser/collab/cloud runners), tracked + non-ignored file import honoring gitattributes, CLI-first.

## Findings from blog posts and talk

Follow-up pass, 2026-10-28. Same caveat as above: WebFetch returns model-made summaries, so quotes are as relayed, not verbatim-certified. "Stated" = in the cited source; "Inference" = mine. Source tags [F]-[K] are defined in Sources.

### Link discovery on [A] https://delta.dev/deltadb [A]

Stated: the page lists four blog posts and one talk: "Software Is Made Between Commits" (zed.dev/blog/introducing-deltadb, Jun 11 2026, Nathan Sobo); "DeltaDB: The Future of Version Control" (/blog/future-of-version-control, Oct 15 2026, Conrad Irwin); "Escaping the tyranny of worktrees" (/blog/escaping-the-tyranny-of-worktrees, Oct 6 2026, Conrad Irwin); "Xanadu Was Waiting for Agents" (zed.dev/blog/agentic-xanadu, Sep 1 2026, Nathan Sobo); and "Watch the talk" (YouTube yif2rday2bw) [A]. Its "Under the hood" line: "DeltaDB is built on CRDTs, loosely based on diamond-types and adapted to work with Git." [A]

### [F] "DeltaDB: The Future of Version Control" (Conrad Irwin), https://delta.dev/blog/future-of-version-control

This is the most technical source. Stated, as relayed by WebFetch:

- Delta schema: each edit carries the inserted text, the character range it applies to, and a "based on" version. A version is either a Git object (commit or blob, identified by SHA-1) or a list of delta IDs (a list because edits can be concurrent). Changes are identified by causal parents, so no distributed clock is needed. [F]
- CRDT: "loosely based on" diamond-types (https://github.com/josephg/diamond-types) but a different implementation so it can interoperate with Git. Unlike Zed's earlier editor CRDT (character IDs), "Instead of baking time into every character's ID, we bake it into the changes." [F]
- Conflict resolution (text, inside DeltaDB): concurrent inserts at the same position are ordered by an actor-ID tiebreak so all replicas converge irrespective of arrival order. Worked example: from "TIVITY", one edit inserts "REA" and another "C" at the same spot; both replicas converge on "REACTIVITY". [F]
- Applying a delta: if it is based on the version you already hold it is applied directly (common case); if based on an older version, a fragment index maps the original position into the current document. [F]
- Storage structure: B-tree fragment index records which delta created each range of characters; immutable trees share structure; each edit copies only the root-to-leaf path, so storage grows with edit size rather than quadratically (the "multi-headed Medusa tree"). On-disk format and backend not described. [F]
- Sync: changes sync "keystroke by keystroke" to the server and collaborators; wire format and transport not described. [F]
- Attribution: each character can record its author: person, agent, or the prompt that produced it. An agent can work on its own copy and merge back in. [F]
- Git: does not replace Git, which is still used for tagging, shipping, deploying ("Git is a _very_ fancy save button"). Changes can be rebased from one Git commit onto another with per-character attribution kept. How commits are created from the delta history is not explained. [F]
- Not covered: API/CLI, rewind UI/mechanics, DeltaDB's own license. [F]

#### [F] verified against the full text (pasted by the user, not via WebFetch)

The user pasted the complete text and four diagrams of [F]. The bullets above match it. Corrections and additions, which are verbatim-sourced:

- Schema is given as "hypothetical Rust": `struct EditDelta { insert: String, at: Range<usize>, based_on: Version }` and `enum Version { Git(Sha1), Delta(Vec<DeltaId>) }`. A delta is "either based on something in Git, like a commit or a specific blob, or on earlier deltas." [F]
- A diagram shows what looks like the real internal types: `HistoryDeltaBatch { id: DeltaBatchId, parent: HistoryVersion, payloads: Vec<HistoryDelta> }` and `DeltaId { timestamp: u64, actor_id: ActorId }`. Deltas are grouped into batches, and each batch has a single parent version. The caption says a delta "carries the text it inserts, the ranges it replaces, and the version it builds on", so it handles replacement, not just insertion. A Git blob is the base for a file's starting text. [F, diagram 1]
- Causal graph: ids are `(Lamport timestamp, actor)`, e.g. `(1,A)`, then concurrent `(2,A)` and `(2,B)`, then `(3,A)` with parents `(2,A)` and `(2,B)`. "Without a shared clock, order comes from the causal graph." [F, diagram 1]
- Tiebreak direction is inconsistent in the source. The prose says "`a` comes before `b`, and you get `REACTIVITY`". The diagram caption says "REA is (4, B), C is (4, A). Same timestamp, so the higher actor goes first", and also yields REACTIVITY. The direction is unspecified, so do not copy either rule as spec. The only requirement is that every replica uses the same deterministic tiebreak. [F]
- Fragment index: "Each file is stored as text plus a B-tree recording which delta created each character." Leaves are fragments with a length and an origin (a delta id, a Git blob, or deleted). Deleted fragments show as zero-length tombstone leaves (`S · 0`, "deleted"), so deletions are retained in the index. [F, diagram 3]
- Cost: "the common case cheap": keystrokes usually arrive in order and nearly every delta is based on the version you are already viewing, so the fragment-index lookup is only needed for stale-based deltas. Structural sharing: the post's own example stores 6 nodes instead of 9 for one appended character. Growth is "linearithmic" in edit size. [F]
- Explicitly not solved: "how to represent the file system. If you rename a directory and I rename another, it's easy to end up with a directory inside itself." [F]
- "delta.dev/docs describes the application built on top of deltadb": DeltaDB is the layer and Delta is the app. [F]

### [G] "Escaping the tyranny of worktrees" (Conrad Irwin), https://delta.dev/blog/escaping-the-tyranny-of-worktrees

Stated:

- Each thread has its own Delta worktree (project files plus recorded history in DeltaDB). A "checkout" is the temporary directory where Delta materialises that worktree on one machine; the worktree is not tied to it, and Delta can remove the checkout afterwards. [G]
- Each workspace has its own Git clone, made cheap with unspecified "some tricks". [G]
- Git: commits are Git commits, branches are Git branches, `origin` is the publish target. Several agents can work on one branch from separate checkouts; "Delta merges their changes before committing and pushing". Standard git switch/merge/rebase/push work in a managed checkout; merge conflicts remain ordinary Git conflicts ("agents don't seem to mind them"). The one-branch-per-worktree restriction does not apply. [G]
- Setup hooks: `.agents/prepare` (committed executable, run from checkout root on mount before the agent starts, must be fast, avoid dependency installs) and `.agents/linked` (git-ignored paths symlinked to shared machine locations). Recommends shared build caches. [G]
- Caveat: checkouts are isolated but share the machine's ports, processes, containers, caches and databases, so projects should make ports/DB names/container names configurable. [G]
- Threads are the durable unit; they can be archived without leaving a directory, and reviewed/edited in a browser without mounting. [G]
- Not covered: rewind, CLI/API, filesystem mount mechanism, storage format, sync internals. [G]

### [H] "Xanadu Was Waiting for Agents" (Nathan Sobo), https://zed.dev/blog/agentic-xanadu

Conceptual essay. Stated:

- Identity: "Every operation by every human and agent is named by an actor plus a Lamport timestamp, forever." Fragments carry stable identities; DeltaDB records which operation produced each fragment and what state it built on. [H]
- Intermediate states are named "by the Git commit it descends from" plus the set of delta IDs applied above it (this matches the version representation in [F]). [H]
- Anchors reference spans that remain resolvable after code moves, unlike line numbers. [H]
- DeltaDB keeps every version by default ("never overwrite, always version"); no retention/engine detail. [H]
- A thread is a live projection of structured history, recomputed rather than stored as a flat document; threads replicate to participants' machines and the browser. [H]
- Git: Delta works with the existing repo; "Every thread is also a Git branch", so non-Delta teammates see a normal repo; files are real files on disk; teams can keep mirroring to GitHub. [H]
- Agents may provision isolated cloud machines (Firecracker-class microVMs), traverse provenance and prior reasoning, and cite what they read. Proposed "landmarks" for frequently referenced files/symbols. [H]
- Not covered: schema, conflict rules beyond convergence, rewind mechanism, sync protocol, DeltaDB license (Zed is described as open source; DeltaDB status not stated). [H]

Note: [G] says commits and branches are plain Git objects while [H] says "every thread is also a Git branch"; the two are compatible but the exact thread-to-branch mapping is not specified (Inference).

### [I] "Software Is Made Between Commits" (Nathan Sobo), https://zed.dev/blog/introducing-deltadb

Stated: deltas are "a stream of fine-grained deltas" capturing every operation between commits, each with stable identity so it can be addressed alone. DeltaDB "embeds conflict-free replicated worktrees" so many people/agents can edit the same files concurrently across machines. A message and the edit it produced are recorded side by side; references anchor to deltas rather than line numbers. From a conversation line you can open the code as it is now or as the agent first wrote it; from a code line you can find the conversation that produced it and later ones that touched it. Agents can read context behind code or convene prior agents. Agents work in real files via a terminal; the full worktree can be mounted to disk. "Git and CI stay for what they're good at." Teammates can join mid-work without committing/pushing. Not covered: schema, storage, sync, API, rewind mechanics (closest: viewing code "as it stood the moment the agent wrote it"), git mechanism, DeltaDB license (footer links to zed-industries/zed but no claim DeltaDB is part of it). [I]

### [J] The talk: "Building a Distributed CRDT Layer for Git" (Conrad Irwin, Zed), https://www.youtube.com/watch?v=yif2rday2bw

- Transcript/slides/description NOT retrieved. WebFetch of the watch page returned only YouTube site chrome; a direct curl for captions (timedtext) and the page description returned nothing; a web search did not locate a transcript, slides or event page.
- Only metadata is confirmed: title and channel come from YouTube oEmbed ("Building a Distributed CRDT Layer for Git - Conrad Irwin, Software Engineer, Zed", channel Zed Industries) [J]; [F] lists it as 22 minutes.
- No talk content is reported here. Contents are unknown; do not infer from the title. Recommend watching it manually or pasting a transcript.

### [K] Supporting first-party pages

- Data storage, https://delta.dev/docs/privacy-and-security/data-storage: "Delta's backend runs entirely on Cloudflare." Repository files, commits and history packs go to R2; thread changes for sync and account settings go to Durable Objects; metadata to KV and D1. Local machine keeps full history of active threads; archived synced threads' local history is removed 3 days after archive/last open (server copy remains, redownloaded on reopen). Encrypted at rest. Self-hosting is not mentioned. [K1]
- Zed CRDT post (editor CRDT, predecessor design), https://zed.dev/blog/crdts: replica ids plus sequence numbers give insertion ids; anchors are (insertion id, offset); inserted text is immutable and fragments split on interior insert; tombstones keep deleted text resolvable, deletions carry a version vector; concurrent inserts at one location sorted descending by Lamport timestamp, replica id as tiebreak; undo via per-operation counters (odd = undone). This is the earlier editor design and [F] says DeltaDB moves time into the changes, so treat it as background, not DeltaDB's spec. [K2]

### Takeaways for a git + coding-agents clone

Inference unless cited.

1. A workable data model is documented: delta = (actor, inserted text, range, based-on version) where version = git object SHA or list of delta IDs; text ordering by Lamport/actor tiebreak [F][H]. File-tree ops are still unspecified.
2. Git integration can be thin: threads/worktrees materialise as separate real clones with ordinary git branches; DeltaDB conflicts are CRDT-resolved, git-level conflicts stay ordinary git conflicts [G][H].
3. Keying intermediate versions to a base git commit plus delta-ID set gives a natural bridge between deltas and commits (and rebase onto another commit) [F][H].
4. Resource isolation beyond files (ports, DBs, containers) is a real concern with parallel agents and is only addressed by convention [G].
5. Cloud architecture reference (R2 + Durable Objects per-thread sync) is a useful design hint for a hosted variant [K1]; a local-first clone is a choice, not a copy.

## Open questions

Status key (added in follow-up pass): ANSWERED / PARTIAL / OPEN, with citations. Original wording retained below each item's status line.

1. Data model details: the exact delta schema, ID scheme, version representation (vector clocks? DAG?), granularity (keystroke vs. save), and how file-tree ops (rename/move/delete) are modelled as CRDT ops.
   - STATUS PARTIAL. Stated: delta = inserted text + char range + based-on version; version = git object SHA-1 or list of delta IDs; causal parents, no distributed clock [F]; ops named by actor + Lamport timestamp [H]; granularity is per keystroke sync [F]. Still open: ID byte format, file-tree op (rename/move/delete) modelling.
2. Where DeltaDB data lives: local store format, server-side storage, what is stored in the cloud vs. on-device (a privacy page exists: https://delta.dev/docs/privacy-and-security/data-storage, unread). Whether it is self-hostable.
   - STATUS PARTIAL. Stated: hosted on Cloudflare (R2 history packs, Durable Objects for sync, KV/D1 metadata); local machine keeps full history of active threads; encrypted at rest; no self-hosting mentioned [K1]. Still open: local store format, self-hostability.
3. Conflict handling: what happens when two threads edit the same lines and are then merged, or when a checkout and worktree diverge; [D] and [E] both explicitly do not describe it.
   - STATUS PARTIAL. Stated: concurrent text inserts converge via actor-ID tiebreak, fragment index maps older-based deltas [F]; git-level merge conflicts remain ordinary git conflicts, agents handle them [G]. Still open: same-line cross-thread merge semantics, checkout vs worktree divergence.
4. Relationship of deltas to git objects: is there any recorded link between a delta range and a resulting commit SHA ("tracing code back to its conversation" is claimed [A] but the mechanism is not).
   - STATUS PARTIAL. Stated: versions reference git commits/blobs by SHA-1 [F]; intermediate states = base commit + delta-ID set [H]; rebase between commits keeps per-char attribution [F]. Still open: how a resulting commit SHA is linked back to a delta range.
5. How ref/branch state is represented inside DeltaDB, and what "forking" a thread does to files and history.
   - STATUS PARTIAL. Stated: commits/branches are plain git [G]; "every thread is also a Git branch" [H]; agent can work on its own copy and merge back [F]. Still open: fork mechanics and internal ref representation.
6. What consistency guarantees apply: "concurrent edits resolve the same way on every machine" [A] is the only claim; nothing on durability, ordering across replicas, offline operation, or partial-failure semantics (beyond [E]'s warnings that multi-worktree merges and landings can partly apply).
   - STATUS PARTIAL. Stated: convergence irrespective of arrival order [F]; all versions kept by default [H]; sync keystroke by keystroke [F]; local history retained for active threads, 3-day pruning for archived synced ones [K1]. Still open: offline semantics, durability/ordering guarantees.
7. Public API/SDK: none is described. The CLI is "Beta" with pages https://delta.dev/docs/cli/get-started, /docs/cli/claude-code, /docs/cli/build-a-plugin (unread); no command surface is documented in the primaries. The only git commands shown are standard git plus `jj git init --colocate`.
   - STATUS OPEN. None of [F][G][H][I] describe API/CLI surface (site nav lists /cli, /docs, /roadmap only [F]).
8. How the file-system mount/sync works (FUSE? file watcher? how external tool edits become deltas).
   - STATUS OPEN (slightly narrowed). Stated: worktree is mounted into a temporary checkout [G]; files are real files on disk [H]; each workspace has its own git clone via unspecified "tricks" [G]. FUSE vs watcher and external-edit capture not described.
9. Size/scale limits, large or binary files, submodules, LFS.
   - STATUS OPEN. Only claim: storage grows with edit size via structural sharing [F]. Binary/LFS/submodules not covered.
10. How "merge worktrees" is implemented (git merge in a checkout vs CRDT merge) and what "thread family" means precisely.
   - PARTIAL. Stated: several agents on one branch from separate checkouts; "Delta merges their changes before committing and pushing" [G]; standard git merge/rebase work in checkouts [G]. Whether this is CRDT merge or git merge, and "thread family", remain unspecified.
11. Licensing/open-source status of DeltaDB; the diamond-types basis is only described as "loosely based".
   - STATUS OPEN. Zed is called open source, but no source states DeltaDB/Delta licence [H][I]; diamond-types is third-party and DeltaDB is a different implementation [F].

## Sources

Read (via WebFetch, summarised by the tool):

- [A] https://delta.dev/deltadb
- [B] https://delta.dev/docs/concepts/delta-and-git
- [C] https://delta.dev/docs/concepts/worktrees
- [D] https://delta.dev/docs/concepts/core-concepts
- [E] https://delta.dev/docs/agents/review-and-sync

Read in follow-up pass (via WebFetch, summarised by the tool):

- [F] https://delta.dev/blog/future-of-version-control
- [G] https://delta.dev/blog/escaping-the-tyranny-of-worktrees
- [H] https://zed.dev/blog/agentic-xanadu
- [I] https://zed.dev/blog/introducing-deltadb
- [J] https://www.youtube.com/watch?v=yif2rday2bw (metadata only via oEmbed https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=yif2rday2bw&format=json; NO transcript retrieved)
- [K1] https://delta.dev/docs/privacy-and-security/data-storage
- [K2] https://zed.dev/blog/crdts

Still linked but not read: https://delta.dev/docs/agents/threads, https://delta.dev/docs/agents/subagents, https://delta.dev/docs/cli/get-started, https://delta.dev/docs/privacy-and-security/data-storage, https://delta.dev/docs/troubleshooting.
