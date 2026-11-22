---
name: edit-source
description: Read, edit, branch and merge source in an Oz IDE repository. Use when the user asks to create a repository, inspect or change files in one, open a proposal, land a merge, or open the repository view.
---

# Working with Oz IDE

Oz IDE (MCP server `oz-ide`) is a source repository built for agents. A snapshot is a commit hash, a branch is a movable name, and a proposal is a pull request. `main` is protected and moves only through the merge queue. The server publishes its full reference as the resource `src://skill`; read it before any workflow not covered here.

## Read

1. `src_repos` lists repositories; `src_create` makes one.
2. `src_outline` returns the headings or symbols of a file with line ranges.
3. `src_read` returns a file, a line range, a `section` or a `symbol`. Keep the `range_hash` or `anchor_hash` it returns.

## Change

1. `src_branch` creates a working branch from the target head. One writer per branch.
2. `src_edit` applies anchored edits to one file: `{ old, new }`, a line range with its `range_hash`, or a `section` or `symbol` with its `anchor_hash`. `src_write` writes, replaces or deletes whole files.
3. Every mutation takes a fresh `mutation_id` (a UUID) and `expect_head` (the branch head you last saw). Reuse a `mutation_id` only to retry a lost response.
4. On `HEAD_MOVED` or `ANCHOR_MISMATCH`, reread only the anchors listed in `failed_edits` at the reported head and retry with a new `mutation_id`. `src_changed_paths` shows what moved.

## Land

1. `src_propose` enqueues the branch on its target and returns a run.
2. `src_proposal_get` shows the run, its queue position and any conflicts. A conflicted proposal is fixed with `src_rebase` onto the target head.
3. `src_merge_finalize` lands the run at the head of the queue.

## View

`src_view` opens the repository view (branches, log, tree, file contents, proposal queue) in hosts that support MCP Apps and returns the same data as structured content elsewhere.
