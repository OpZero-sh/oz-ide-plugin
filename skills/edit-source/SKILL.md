---
name: edit-source
description: Read and change source files in an Oz IDE repository on a working branch, then open a proposal. Use when someone says "create a repository", "add a file to my repo", "change this function", "rename this heading in the README", "make a branch and edit", "open a proposal", "open a pull request in Oz IDE", or asks to see the repository view.
---

# Read and edit source in Oz IDE

Oz IDE (MCP server `oz-ide`) is a private source repository built for assistants. A snapshot is a commit hash. A branch is a movable name. A proposal is the equivalent of a pull request. `main` is protected: it changes only when a proposal is merged. Every repository belongs to the signed in user and is invisible to anyone else.

This skill covers the everyday loop: find the code, branch, edit, propose. Merging and conflicts are in `review-and-merge`. History and undo are in `history-and-restore`. Whole folders and large files are in `import-and-export`.

## When to use

- The user wants a new repository, or a file added, changed or deleted in one.
- The user wants a change prepared for review.

Do not edit when the user asked only for an explanation or a plan. Read, answer, and stop. For a wide read across many files, delegate to the `source-navigator` agent.

## Rules that prevent lost work

1. Read before you write, and edit against what you read. Keep the `range_hash` or `anchor_hash` that `src_read` returns and send it back with the edit.
2. Every write takes `expect_head`: the commit you last saw on that branch. Take it from `src_branch` (`head`), from `src_branches`, or from `commit` in the result of your last write.
3. Every write takes a fresh `mutation_id` (a UUID). Reuse one only to retry a call whose response you never received. The retry returns the original result.
4. One writer per branch. If another assistant or person is working in the repository, use your own branch.
5. Never try to write to `main` directly. Work on a branch and propose.

Details, rebase behavior and every refusal code are in [references/anchors-and-retries.md](references/anchors-and-retries.md). The full tool list with what each tool changes is in [references/tool-index.md](references/tool-index.md).

## Steps

### 1. Find the repository and the code

1. `src_repos` lists the user's repository ids. If the user wants a new one, `src_create` with `repo` (lowercase letters, digits and hyphens, at most 23 characters).
2. `src_list` with `repo` and `prefix` shows a directory. `depth` goes deeper than one level.
3. `src_outline` with `repo` and `path` returns the headings of a Markdown file or the top level symbols of a code file, each with its line range.
4. `src_read` returns the file, or just a `section` (a Markdown heading), a `symbol` (a top level definition) or `start_line` to `end_line`. Output is capped at 16 KB by default. Raise `max_bytes` to read a larger body whole. `src_read_many` reads up to 50 small files in one call.

Reads default to the head of `main`. Pass `ref: { branch: "..." }` to read your working branch.

### 2. Make a working branch

`src_branch` with `repo`. It branches from the head of `main` unless you pass `from`, and is named `contrib/<user>/<n>` unless you pass `name`. Keep the `head` it returns as your first `expect_head`.

### 3. Change files

Use `src_edit` to change part of one text file. Each entry in `edits` names its target in one of four ways:

| Form | Use for |
|---|---|
| `{ old, new }` | A short exact string that occurs once in the file. |
| `{ start_line, end_line, range_hash, new }` | The exact lines you read, with the `range_hash` from that read. |
| `{ section, anchor_hash, new }` | The body of a Markdown section. The heading line stays. |
| `{ symbol, anchor_hash, new }` | The body of a top level symbol. |

Use `src_write` to create a file, replace a whole file or delete one. Each entry in `files` has `path` and `content` (or `content_b64` for binary), or `delete: true`. `src_edit` cannot create a file: a missing path is refused with `PATH_NOT_FOUND`.

Both tools need `repo`, `branch`, `message`, `mutation_id` and `expect_head`. Each successful write returns `commit`: that is the new head and your next `expect_head`. An edit that leaves the file identical returns `noop: true` and writes no commit.

### 4. Open a proposal

`src_propose` with `repo`, `branch` and a `summary` of what changed. The target is `main` unless you pass `target`. The result gives the proposal id, its queue `position` and a `run` with any `conflicts`. A proposal that would change nothing is refused with `EMPTY_CANDIDATE`.

Stop here unless the user asked you to merge. Tell them the proposal id and what it contains.

### 5. Show the repository

`src_view` with `repo` opens the repository view in hosts that support MCP Apps: files, history, branches and the proposal queue. Other hosts get the same data as structured content. It changes nothing.

## When a write is refused

- `HEAD_MOVED`: the branch moved since `expect_head`. The response lists `expected`, `actual`, `overlapping_paths` and, for `src_edit`, each failed edit. Reread only those parts at `actual`, then retry with the new hashes, the new `expect_head` and a new `mutation_id`.
- `ANCHOR_MISMATCH`: the text or hash you sent does not match the snapshot you named. Reread the file and rebuild the edit. Widen an `old` string that matches more than once.
- A protected branch refuses direct writes. Create a branch and propose.

## What to tell the user

The repository and branch, each file changed with a one line description, the commit hash of the last write, and the proposal id if you opened one. Say plainly if you stopped before proposing or merging.

## Examples

"Create a repository called notes and add a README that describes the project."
`src_create` with `repo: "notes"`. `src_branch`. `src_write` with one file. `src_propose`. Because `main` is protected, the README reaches `main` only when the proposal is merged, so ask whether to merge it.

"In my site repository change the heading in index.html to Welcome."
`src_read` the file on `main`, `src_branch`, `src_edit` with `{ old, new }` on the heading line, `src_propose`. Report the proposal id.

"Explain how parseConfig is called."
Read only. `src_outline`, `src_read` with `symbol`, then answer. No branch, no edit.

## Limits

- Up to 25 repositories per user.
- Inline file content up to 1 MB per file. Larger files need the upload flow in `import-and-export`.
- `src_edit` works on text files stored inline. Binary files and files over 1 MB are replaced whole with `src_write`.
- Symbols are found by pattern matching on top level definitions. Nested functions and unusual syntax may not be listed. Fall back to a line range.
- The server publishes its own reference as the resource `src://skill`. Read it when a case is not covered here.
