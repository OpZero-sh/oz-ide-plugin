---
name: review-and-merge
description: Review Oz IDE proposals, resolve conflicts, and merge or reject them through the merge queue. Use when someone says "what proposals are open", "show me what this proposal changes", "merge it", "land the first one in the queue", "my proposal has a conflict", "rebase my branch", or "reject that proposal".
---

# Review, merge and reject proposals

In Oz IDE (MCP server `oz-ide`) a proposal asks to merge a branch into a target, normally `main`. Proposals wait in a merge queue per target. Each one has a merge run: a trial merge against the result of the proposal ahead of it, or against the target head when it is first in line. Only the run at the head of the queue can be merged.

## When to use

- The user wants to see open proposals or what one would change.
- The user wants a proposal merged or rejected.
- A proposal is in conflict and needs its branch brought up to date.

Merging moves a protected branch. Rejecting closes a proposal. Do either only when the user asked for that specific proposal. To have a proposal read closely before a decision, delegate to the `proposal-reviewer` agent.

## Review

1. `src_proposals` with `repo`. Filter with `state` (`queued`, `conflict`, `rejected`, `merged`), `target` or `mine: true`. With `target` the rows come back in queue order with a `position`.
2. `src_proposal_get` with `repo` and `proposal` (the id). It returns `source`, `target`, `state`, `summary`, `position` and the current `run`: its `id`, `source_head`, `target_head`, `merge_base`, `candidate_root` and `conflicts`.
3. See the change. `src_diff` with `from: { branch: <target> }`, `to: { branch: <source> }` and `text: true` returns a unified diff. Without `text` it returns added and removed line counts per path. Use `prefix` to limit it to a directory.
4. Read the merged result when it matters. `src_read_many` with `run` set to the run id returns the files as that run would merge them.
5. Summarize for the user: what changes, which files, queue position, and any conflicts.

## Merge

1. Read the proposal again with `src_proposal_get` just before merging. A run id can change when the queue ahead of it changes, so do not reuse an old one.
2. Confirm it is first in its queue and that `conflicts` is empty. `src_proposals` returns `queues`, the proposal ids in order for each target. The first id is the head.
3. Call `src_merge_finalize` with `repo`, `run` (the run id, not the proposal id) and a fresh `mutation_id`. Pass `expect_target_head` and `expect_candidate_root` from the run you reviewed. The merge is then refused if the target moved or the merged result is not the one you looked at.
4. The rest of the queue is evaluated again in the same call. Report the merged commit and what is now first in line.

If the user asks to merge a proposal that is not first, say what is ahead of it and ask whether to merge those first. Do not reject other people's proposals to clear the way.

## Resolve a conflict

A proposal in `conflict` has left the queue. It comes back on its own, at the tail and with the same id, once its branch merges cleanly.

1. `src_proposal_get` to see `conflicts` and the run's `target_head`.
2. Get the branch head with `src_branches`.
3. `src_rebase` with `repo`, `branch`, `message`, a fresh `mutation_id`, `expect_head` (the branch head) and `onto` (the target head). This replays the branch's changes as one new commit on top of the target.
4. If it is refused with `REBASE_CONFLICT`, the response lists `overlapping_paths`: files changed on both sides. Show them to the user and ask which side wins for each case. Then either:
   - `overlap: "branch"` keeps the branch version of those paths, or
   - `overlap: "onto"` takes the target version, after which you reapply the intended change with `src_edit` on the new head.
   Choosing a side drops the other side's change to those files. Never choose without the user's answer.
5. `src_proposal_get` again to confirm the proposal is back in the queue.

## Reject

`src_reject` with `repo`, `proposal` and a `reason`. The proposal is closed. Its branch and commits are kept, so the work is not lost. Proposals behind it are evaluated again.

## Problems

- `CANDIDATE_MISMATCH` or a refused finalize: the queue or target changed after you reviewed. Read the proposal again, review the new run, and ask before merging.
- `PROPOSAL_NOT_FOUND`: list proposals and check the id.
- `HEAD_MOVED` on `src_rebase`: the branch moved since `expect_head`. Read `src_branches` again and retry with a new `mutation_id`.
- A merged result is consistent as text. Nothing in Oz IDE runs the code, so say that tests were not run unless the user ran them elsewhere.

## Examples

"Show me what the open proposals in site would change, then merge the first one in the queue."
`src_proposals` with `target: "main"`. For each, `src_diff` with `text: true` and a short summary. Then `src_proposal_get` on the first, and `src_merge_finalize` with its run id and the two `expect` values. Report the merged commit.

"My proposal says conflict."
`src_proposal_get`, `src_branches`, `src_rebase` onto the target head. If `REBASE_CONFLICT`, list the overlapping files and ask which side to keep.

"Reject proposal 4, it is superseded."
`src_reject` with `proposal: 4` and that reason. Say that the branch is still there.

## What to tell the user

For a review: proposal id, source and target, files changed, conflicts, queue position. For a merge: the merged commit and the new head of the target. For a rebase: the new branch head and whether any side was dropped.
