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

## Open questions

Not stated in the pages read (so not guessed):

1. Data model details: the exact delta schema, ID scheme, version representation (vector clocks? DAG?), granularity (keystroke vs. save), and how file-tree ops (rename/move/delete) are modelled as CRDT ops.
2. Where DeltaDB data lives: local store format, server-side storage, what is stored in the cloud vs. on-device (a privacy page exists: https://delta.dev/docs/privacy-and-security/data-storage, unread). Whether it is self-hostable.
3. Conflict handling: what happens when two threads edit the same lines and are then merged, or when a checkout and worktree diverge; [D] and [E] both explicitly do not describe it.
4. Relationship of deltas to git objects: is there any recorded link between a delta range and a resulting commit SHA ("tracing code back to its conversation" is claimed [A] but the mechanism is not).
5. How ref/branch state is represented inside DeltaDB, and what "forking" a thread does to files and history.
6. What consistency guarantees apply: "concurrent edits resolve the same way on every machine" [A] is the only claim; nothing on durability, ordering across replicas, offline operation, or partial-failure semantics (beyond [E]'s warnings that multi-worktree merges and landings can partly apply).
7. Public API/SDK: none is described. The CLI is "Beta" with pages https://delta.dev/docs/cli/get-started, /docs/cli/claude-code, /docs/cli/build-a-plugin (unread); no command surface is documented in the primaries. The only git commands shown are standard git plus `jj git init --colocate`.
8. How the file-system mount/sync works (FUSE? file watcher? how external tool edits become deltas).
9. Size/scale limits, large or binary files, submodules, LFS.
10. How "merge worktrees" is implemented (git merge in a checkout vs CRDT merge) and what "thread family" means precisely.
11. Licensing/open-source status of DeltaDB; the diamond-types basis is only described as "loosely based".

## Sources

Read (via WebFetch, summarised by the tool):

- [A] https://delta.dev/deltadb
- [B] https://delta.dev/docs/concepts/delta-and-git
- [C] https://delta.dev/docs/concepts/worktrees
- [D] https://delta.dev/docs/concepts/core-concepts
- [E] https://delta.dev/docs/agents/review-and-sync

Linked but not read (candidates for follow-up): https://delta.dev/blog/future-of-version-control, https://delta.dev/blog/escaping-the-tyranny-of-worktrees, https://zed.dev/blog/introducing-deltadb, https://zed.dev/blog/agentic-xanadu, https://www.youtube.com/watch?v=yif2rday2bw, https://delta.dev/docs/agents/threads, https://delta.dev/docs/agents/subagents, https://delta.dev/docs/cli/get-started, https://delta.dev/docs/privacy-and-security/data-storage, https://delta.dev/docs/troubleshooting.
