---
name: history-and-restore
description: Read the history of an Oz IDE repository, compare snapshots, undo changes on a branch, and manage branch protection. Use when someone says "show the history of this file", "what changed since yesterday", "diff these two branches", "who changed this", "undo my last change", "restore the branch to an earlier commit", "list my branches", or "protect this branch".
---

# History, comparison and restore

Oz IDE (MCP server `oz-ide`) keeps every commit. Replaced and deleted files stay reachable from earlier commits, so most mistakes on a branch can be undone. This skill reads history and uses it.

## When to use

- The user wants to know what changed, when, and in which commit.
- The user wants two branches or commits compared.
- The user wants a working branch put back to an earlier state.
- The user asks about branches or branch protection.

## Read history

| Question | Call |
|---|---|
| What branches exist and where do they point | `src_branches` with `repo`. Each row has `name`, `head`, `protected`, `owner` and when it last moved. |
| What happened on a branch | `src_log` with `repo`, optional `ref`, `limit` (up to 100). Newest first. |
| What happened to one file | `src_log` with `path`. Only commits that changed that path. |
| More history | `src_log` with `before` set to the last commit hash you saw. |
| What differs between two snapshots | `src_diff` with `from` and `to`. Add `text: true` for a unified diff, `prefix` to limit it to a directory, `context_lines` for more or less context. |
| Which paths differ, without content | `src_changed_paths` with `from` and `to`. |
| A file as it was | `src_read` with `ref: { snapshot: "<commit>" }`. |

`ref`, `from` and `to` are objects: `{ branch: "name" }` or `{ snapshot: "<commit hash>" }`, with at least 12 characters of the hash. Each defaults to the head of `main`.

By default `src_log` follows first parents, so a branch that receives merges shows one entry per merged proposal. Pass `first_parent: false` to see every commit.

When you report history, give the commit hash (12 characters is enough), the message and the time for each entry. Do not invent an author: report the author value the log returns, which is an account identifier.

## Restore a branch to an earlier commit

`src_restore` adds one new commit whose files equal an earlier commit. It does not rewrite history. The versions it overwrites remain reachable.

1. Find the commit that made the unwanted change with `src_log` (add `path` when the user names a file). The target is the commit just before it on the branch: call `src_log` on the branch without `path` and with `before` set to the unwanted commit, and take the first entry. Show the user both commits and confirm the target.
2. Show what will be lost: `src_diff` from the target commit to the branch head, with `text: true` when it is short.
3. Get the branch head from `src_branches` for `expect_head`.
4. Call `src_restore` with `repo`, `branch`, `message`, a fresh `mutation_id`, `expect_head` and `to` (the commit hash).
5. Confirm with `src_log` and report the new head.

`src_restore` works on a branch you can write to. `main` is protected, so to undo something that already landed there: create a branch, restore that branch to the earlier commit, and open a proposal (see `edit-source` and `review-and-merge`).

To undo only one file, do not restore the whole branch. Read the old version with `src_read` at the earlier commit and write it back with `src_write`.

## Branch protection

A protected branch refuses direct writes and changes only when a proposal that targets it is merged. `main` is protected when a repository is created.

`src_protect` with `repo`, `branch` and `protected` (`true` or `false`) sets or clears it. Clearing protection on `main` allows direct writes to it, which removes the review step for everyone working in that repository. Do it only when the user asks in so many words, say what it means before you call it, and offer to protect the branch again when they are done.

## Problems

- `HEAD_MOVED` on `src_restore`: the branch moved since `expect_head`. Read `src_branches` again and retry with a new `mutation_id`.
- A commit hash that is not unique or not found: ask for more characters, or list with `src_log`.
- An empty log for a `path`: the path may be misspelled or may never have existed on that branch. Check with `src_list`.

## Examples

"Show the history of docs/guide.md and restore my working branch to the commit before the last change."
`src_log` with `path: "docs/guide.md"` and `ref: { branch: <working branch> }`. Show the entries. The first entry is the last change. Call `src_log` on the branch with `before` set to that commit to find the commit before it. Show `src_diff` from that commit to the head, confirm with the user, then `src_restore`. If other files changed in between, say that restoring the branch reverts them too and offer to restore only the one file.

"What changed on main this week?"
`src_log` on `main`. List the entries dated within the week. For detail on one, `src_diff` between that commit and the one before it.

"Compare my branch with main."
`src_diff` with `from: { branch: "main" }` and `to: { branch: <branch> }`. Give line counts per path first, and the unified diff for the files the user asks about.

## Limits

- History shows what was committed and by which account. It does not record why beyond the commit message.
- A restore is a new commit, so it can itself be undone the same way.
