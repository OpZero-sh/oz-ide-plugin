# Files larger than 1 MB

Oz IDE refuses inline content over 1 MB per file with `PAYLOAD_TOO_LARGE`. Larger files, up to 25 MB, are uploaded to storage first and then named by hash in a commit. Taken from the server's reference (`src://skill`) and the `src_upload_prepare` schema.

## The hash

The id of a file's content is the SHA-256 of a Git style blob object: the text `blob <size>` followed by one zero byte, then the file's bytes, where `<size>` is the content length in bytes as a decimal number. The result is 64 hexadecimal characters. The upload tools need all 64.

## Steps

1. Compute the hash and the size of each file.
2. Optional: call `src_blob_missing` with `repo` and `hashes` to learn which contents the repository does not hold yet. Skip the ones it already has.
3. Call `src_upload_prepare` with `repo` and `blobs`, a list of `{ hash, size }` (up to 200). For each entry the answer is either `already_present: true`, or a `url`, the `headers` to send and an `expires` time 15 minutes away. No branch changes.
4. Upload each file with an HTTP PUT to its `url`, sending the returned `headers` exactly as given. The body is the blob object, not the bare file: the same `blob <size>` text and zero byte as in the hash, followed by the bytes. Storage refuses a body whose SHA-256 differs from the hash with HTTP 400.
5. Commit by hash. Use `blob_hash` on the file entry in `src_write` or `src_ingest`, or `hash` in `src_commit_manifest`. The server checks that the object exists at the expected size before the commit.

## After the upload

| Situation | What happens |
|---|---|
| A referenced hash has no object | `BLOB_NOT_STAGED` with the `hashes`. Nothing is committed. |
| The upload address expired | Call `src_upload_prepare` again. |
| The upload was never committed | It is removed after 24 hours. |
| `src_blob_missing` on a prepared but not uploaded hash | Reported as missing and also listed under `staged`. |
| `src_read` on a large file | Returns `size`, the full `blob_hash` and a `resource` link of the form `src://<repo>/<commit>/<path>`, with no content. Read the link as an MCP resource to get the bytes. |
| `src_edit` on a large file | Refused with `BLOB_NOT_INLINE`. Replace the whole file. |
| `STORAGE_UNAVAILABLE` with `retryable: true` | Storage could not be reached during the check. Retry with the same `mutation_id`. |

## Care

- The upload address is a short lived signed URL. Do not print it in the conversation or store it.
- Tell the user which files are being uploaded and their sizes before you start.
- Files over 25 MB are refused with `BLOB_TOO_LARGE`. Say so and suggest keeping such files outside the repository.
