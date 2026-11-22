# Anchors, compare and swap, and retries

How Oz IDE keeps a write from landing on content that changed since it was read. Taken from the server's own reference (`src://skill`) and its tool schemas.

## The two guards on every write

| Argument | What it is | Rule |
|---|---|---|
| `expect_head` | The commit you last saw as the head of the branch | Take it from `src_branch`, `src_branches` or the `commit` of your last write. A 12 character prefix is accepted. |
| `mutation_id` | A unique id you generate, such as a UUID, at least 8 characters | New for every attempt. Reuse it only to retry a call whose response was lost. That retry returns the original result word for word. |

Every write also takes `repo`, `branch` and `message`.

## Hashes returned by `src_read`

| Hash | Returned for | Send it back as |
|---|---|---|
| `range_hash` | The exact lines returned | `range_hash` on a line range edit |
| `anchor_hash` | The exact body bytes of a `section` or `symbol` read | `anchor_hash` on a section or symbol edit |

Hashes come back as 12 character prefixes. `full_hashes: true` returns all 64 characters. A body larger than `max_bytes` comes back as lines with no `anchor_hash`. Raise `max_bytes` to read it whole.

A section body excludes its heading line and includes the blank lines around it. A symbol body runs through the newline that ends its last line. `src_edit` replaces exactly those bytes with `new`, so include the trailing newline you want.

Always send the `anchor_hash` you read. It lets the edit be applied over a file someone else changed elsewhere, and it turns a body that changed under you into a refusal instead of an overwrite.

## What happens when the branch head moved

The `rebase` argument sets how far the server may go. Leave it out to get the default.

| Tier | Default for | Behavior |
|---|---|---|
| `path` | `src_write`, `src_ingest`, `src_restore` | The write applies if none of the paths it touches changed since `expect_head`. The response carries `rebased: "path"` and `rebased_onto`. |
| `edit` | `src_edit` | As `path`, and edits on a changed file are tried again against the current content: an `old` and `new` edit is relocated by its unique match, a section or symbol edit applies only while its `anchor_hash` still matches, and a line range edit is refused. |
| `false` | none | Never rebase. Any head move is refused. |

A rebased result is consistent as text. It is not proof that the code still builds.

## Refusal codes

A refusal changes nothing.

| Code | Meaning | What to do |
|---|---|---|
| `HEAD_MOVED` | The branch head is not `expect_head` and the write could not be rebased. Carries `expected`, `actual` and `overlapping_paths`. For `src_edit` it also carries `path`, `failed_edits` and `failed_edit_reasons`. | Call `src_changed_paths` from `expected` to `actual` to see what moved. Reread only the failed anchors at `actual`. Retry with the new hashes, `expect_head: actual` and a new `mutation_id`. |
| `ANCHOR_MISMATCH` | An anchor does not match exactly once at `expect_head`, or a hash does not name the content there. | The edit was built from the wrong text. Reread and rebuild it. Add surrounding text to an `old` string that matches twice. |
| `ANCHOR_CHANGED` | The `anchor_hash` you sent no longer names the current body. | Someone changed that section or symbol. Reread it and decide whether your change still applies. |
| `ANCHOR_UNVERIFIED` | A section or symbol edit without `anchor_hash` on a file that changed since `expect_head`. | Reread to get the hash and send it. |
| `PATH_NOT_FOUND` | `src_edit` on a path that does not exist. | Create the file with `src_write`. |
| `BINARY` | `src_edit` on a binary file. | Replace it with `src_write` and `content_b64`. |
| `BLOB_NOT_INLINE` | `src_edit` on a file stored out of line (over 1 MB). | Replace the whole file with `src_write` and `blob_hash`. |
| `PAYLOAD_TOO_LARGE` | Inline content over 1 MB in one file. | Use the upload flow in the `import-and-export` skill. |
| `DUPLICATE_PATH` | The same path twice in one `src_write`. | Send each path once. |
| `REPO_EXISTS`, `REPO_LIMIT` | `src_create` with an id already in use, or past 25 repositories. | Pick another id, or tell the user they are at the limit. |
| `EMPTY_CANDIDATE` | `src_propose` on a branch that would change nothing on the target. | Nothing to propose. Check that your writes went to this branch. |

Each entry in `failed_edit_reasons` has `index`, `reason` and `anchor`, so you can tell which edit in the list failed and why.

## Cheap rereads

- `unchanged_since` on `src_read`: pass a commit hash. If the file is identical there, the answer is `unchanged: true` with no content.
- `src_changed_paths` with `from` and `to`: the paths that differ between two snapshots, without content.

## One mode per branch

Work on a branch either through `src_edit` and `src_write`, or through a checked out folder that you send back with `src_ingest`. Do not mix the two on one branch: an ingest that overlaps a later edit is refused with `HEAD_MOVED`.
