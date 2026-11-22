---
name: import-and-export
description: Move whole folders and large files in and out of an Oz IDE repository, back a repository up, and check its usage. Use when someone says "import this folder into Oz IDE", "commit my working directory", "check out the repository to disk", "sync my local changes", "upload a large file", "export or back up my repository", or "how much space is my repo using".
---

# Import, check out, export and usage

Most work in Oz IDE (MCP server `oz-ide`) goes through single file edits (see `edit-source`). This skill covers the bulk paths: committing a whole set of files at once, reproducing a snapshot as files on disk, files over 1 MB, raw export, and usage figures.

## When to use

- The user has a folder of files to put into a repository in one commit.
- The user wants the repository's files on disk, or wants local changes sent back.
- A file is larger than 1 MB.
- The user wants a backup or wants to know usage and limits.

Use one mode per branch. A branch that you fill with `src_ingest` from a folder should not also be changed with `src_edit`, and the other way round. Mixing them leads to `HEAD_MOVED` refusals.

## Commit a whole folder

1. Pick or create a working branch (`src_branch`). `main` is protected, so a bulk import still lands through a proposal.
2. Read every file you intend to send. Check for credentials and for files the user would not want stored, such as dependency folders and build output. Ask before including anything doubtful.
3. Call `src_ingest` with `repo`, `branch`, `message`, a fresh `mutation_id`, `expect_head` and `files`: one entry per file with `path` and `content` (or `content_b64` for binary).
   - The set is compared with the branch at `expect_head` and only the differences are committed.
   - Files that exist on the branch but are missing from your set are kept, unless you pass `prune: true`. Pass it only when the user wants the branch to match the folder exactly, and tell them which paths will be deleted first (`src_checkout_manifest` shows what is there now).
   - `prefix` limits the whole operation to one directory.
   - A file that has not changed can be sent as `blob_hash` instead of its bytes.
4. Keep `commit` from the result as the next `expect_head`, then `src_propose`.

## Reproduce a snapshot on disk

`src_checkout_manifest` with `repo` and `ref` returns the commit, the root hash and every path with its content hash, mode and size. Read the contents with `src_read_many` (up to 50 small files per call) and write them to disk with your own file tools. The manifest holds up to 5,000 paths per call and sets `truncated` beyond that. Use `prefix` to take one directory at a time.

Record the commit you checked out. It is the `expect_head` for sending changes back with `src_ingest`. The manifest returns it as `src_head`, and the server's reference keeps that value in a file named `.src/HEAD` at the top of the checkout. Do not run `git init` inside a folder checked out this way: an Oz IDE checkout is not a Git working tree.

## Files over 1 MB

Inline content is limited to 1 MB per file. A larger file, up to 25 MB, is uploaded separately and then referenced by hash. This needs a client that can compute a SHA-256 and send an HTTP PUT. The steps are in [references/large-files.md](references/large-files.md). Tell the user before you upload: the file leaves their machine and goes to OpZero's storage at an address the server returns.

Reading works the same way in reverse: `src_read` on a large file returns its size, hash and a `src://` resource link instead of content. Read that link as an MCP resource to get the bytes.

## Export a repository

`src_export` with `repo` returns the raw contents for backup or transfer: branches with full head hashes, commits in order with their stored objects, and objects by full hash. Page the commit list with `after` and `limit` (up to 500 per page). Proposals, the merge queue and write receipts are not part of an export.

For a plain copy of the current files, a checkout (above) is simpler than an export. Offer the export when the user wants history too.

## Usage and limits

`src_quota_get` with `repo` returns object counts and stored bytes, write and rebase counters, queue depths and the limits that apply. Quote the figures it returns.

## Problems

- `PAYLOAD_TOO_LARGE`: a file in the set is over 1 MB inline. Upload it first and send it as `blob_hash`.
- `BLOB_NOT_STAGED`: a hash you referenced has no uploaded object. The response lists the hashes. Upload them and retry.
- `HEAD_MOVED` on `src_ingest`: paths in your set changed on the branch since `expect_head`. Use `src_changed_paths` to see which, merge those files by hand, and retry with a new `mutation_id`.
- `STORAGE_UNAVAILABLE` with `retryable: true`: storage could not be reached. Retry with the same `mutation_id`.

## Examples

"Import this folder as a new repository called site."
`src_create`, `src_branch`, read the folder, leave out anything that looks like a secret or a build artifact and say so, `src_ingest`, `src_propose`. Ask whether to merge.

"Check out my api repository so I can run it locally."
`src_checkout_manifest`, then `src_read_many` in batches, writing each file to the folder the user names. Report the commit that was checked out.

"How close am I to the limits on notes?"
`src_quota_get` and report the figures next to the limits it returns.

## Limits

- Up to 25 MB per file through upload. Larger files are refused with `BLOB_TOO_LARGE`.
- An upload address expires after 15 minutes. Uploaded files that no commit references within 24 hours are removed.
- An export is raw objects, not a ZIP of files.
