# Oz IDE tool index

Every tool the `oz-ide` server lists, grouped by effect as the server's own page does. Read tools change nothing. Tools under "Adds" create new content only. Tools under "Changes or removes" can replace or delete content, move a branch, or change its rules, and a client may ask the user to confirm them.

## Read

| Tool | What it does | Covered in skill |
|---|---|---|
| `src_view` | Opens the repository view: files, history, branches and the proposal queue. | `edit-source` |
| `src_repos` | Lists the user's repositories. | `edit-source` |
| `src_branches` | Lists branches with head commits and protection. | `history-and-restore` |
| `src_list` | Lists files and directories at a branch or commit. | `edit-source` |
| `src_read` | Reads a file, or one line range, Markdown section or code symbol of it. | `edit-source` |
| `src_read_many` | Reads up to 50 small files in one call. | `edit-source` |
| `src_outline` | Returns the headings or top level symbols of a file. | `edit-source` |
| `src_log` | Shows commit history. | `history-and-restore` |
| `src_diff` | Compares two branches or commits. | `history-and-restore`, `review-and-merge` |
| `src_changed_paths` | Lists the paths that differ between two branches or commits. | `history-and-restore` |
| `src_checkout_manifest` | Returns every path of a snapshot with its hash, mode and size. | `import-and-export` |
| `src_blob_missing` | Reports which file contents the repository does not hold yet. | `import-and-export` |
| `src_proposals` | Lists proposals and their queue order. | `review-and-merge` |
| `src_proposal_get` | Returns one proposal with its merge run and conflicts. | `review-and-merge` |
| `src_export` | Exports the raw contents of a repository. | `import-and-export` |
| `src_quota_get` | Returns usage figures and limits for a repository. | `import-and-export` |

## Adds

| Tool | What it does | Covered in skill |
|---|---|---|
| `src_create` | Creates a new, empty repository with a protected `main`. | `edit-source` |
| `src_branch` | Creates a branch at an existing commit. | `edit-source` |
| `src_propose` | Opens a proposal to merge a branch. | `edit-source` |
| `src_upload_prepare` | Prepares uploads for files larger than 1 MB. | `import-and-export` |

## Changes or removes

| Tool | What it does | Covered in skill |
|---|---|---|
| `src_write` | Writes, replaces or deletes whole files in one commit. | `edit-source` |
| `src_edit` | Changes part of one file in one commit. | `edit-source` |
| `src_restore` | Sets a branch back to the content of an earlier commit. | `history-and-restore` |
| `src_rebase` | Moves a branch onto another commit. | `review-and-merge` |
| `src_ingest` | Commits a full set of files, applying only the differences. | `import-and-export` |
| `src_commit_manifest` | Commits a list of paths by content hash. | `import-and-export` |
| `src_reject` | Rejects a proposal and removes it from the queue. | `review-and-merge` |
| `src_merge_finalize` | Merges a proposal into its target branch. | `review-and-merge` |
| `src_protect` | Sets or clears protection on a branch. | `history-and-restore` |

## References

Read tools take `ref` as an object: `{ branch: "main" }`, or `{ snapshot: "<commit hash>" }` with at least 12 characters of the hash. It defaults to the head of `main`, and the response names the commit it resolved to as `resolved_snapshot`. `src_view` is the exception: its `ref` is a plain string.

## Limits

- Up to 25 repositories per user. A repository id is lowercase letters, digits and hyphens, at most 23 characters.
- Inline file content up to 1 MB per file. Larger files up to 25 MB through a prepared upload.
- `src_read` returns 16 KB by default and marks truncated output.
- History is kept. Replaced and deleted files remain reachable from earlier commits.
