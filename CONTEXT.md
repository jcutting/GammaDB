# GammaDB

A version-control layer that sits alongside git and records every change made by humans and coding agents as it happens, so parallel agents and people can work on the same repo without stepping on each other and every change carries provenance. Git stays the system of record for commits.

## Language

**Actor**:
A human or a coding agent that writes changes. Every change is attributed to one Actor.
_Avoid_: User, author, client, participant

**Delta**:
One recorded change by an Actor, at character level: the text it inserts, the range it replaces, and the Version it builds on.
_Avoid_: Edit, patch, diff, operation

**Version**:
A point a Delta can build on: either a git object (a commit or a blob) or a set of earlier Deltas.
_Avoid_: Revision, snapshot, state

**Thread**:
The durable unit of work: one conversation or task, owned by a human, in which Actors make Deltas. Each Thread has its own Worktree.
_Avoid_: Session, task, branch

**Worktree**:
The history of one Thread's project files, kept by GammaDB. Not a git worktree.
_Avoid_: Workspace, git worktree

**Checkout**:
A directory on one machine where a Worktree is materialised so tools and editors can read and write files. Each Checkout has its own git clone.
_Avoid_: Working copy, clone, workspace

**Land**:
The explicit act of turning a Thread's Deltas into a git commit. Nothing is committed implicitly.
_Avoid_: Commit (for the GammaDB action), sync, publish, push

**Presence**:
Which Actors are currently in a file and where their cursors are, shown live to the other Actors in the same Thread.
_Avoid_: Awareness, status

## Relationships

- An **Actor** makes **Deltas** inside a **Thread**; a **Thread** has exactly one **Worktree** and may be materialised as several **Checkouts**.
- A **Delta** builds on a **Version**; **Landing** a **Thread** produces a git commit authored by the **Thread**'s owning human, with `Co-Authored-By` trailers for each agent **Actor**.
