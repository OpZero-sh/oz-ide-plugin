# Oz IDE

Inspect a source tree and make a change against the version you actually read. Branches, anchored edits and a merge queue help multiple contributors work on one repository. Repositories are private to the account that created them.

[Connect with Claude, ChatGPT, Muse Code, or Grok](PROVIDERS.md)

> “Find the function that formats our report, explain its callers, and propose the smallest change to include the date.”

> “Show me what the open proposals in site would change, then merge the first one in the queue.”

> “Show the history of docs/guide.md and restore my working branch to the commit before the last change.”

## What is in the plugin

### Skills

| Skill | What it does |
|---|---|
| `edit-source` | Creates a repository, finds code by outline, section or symbol, edits on a working branch against the exact text that was read, and opens a proposal. |
| `review-and-merge` | Lists proposals, shows what each would change, brings a conflicted branch up to date, and merges or rejects through the merge queue. |
| `history-and-restore` | Reads commit history, compares branches and commits, restores a branch to an earlier commit, and manages branch protection. |
| `import-and-export` | Commits a whole folder at once, reproduces a snapshot as files, handles files over 1 MB, exports a repository and reports usage. |

Skill folders hold `references` files with the edit anchoring rules, refusal codes, the full tool index and the large file procedure. Claude reads them when it needs them.

### Agents

Both agents can only read.

- `source-navigator` traces a behavior through the smallest set of files and returns where it lives, how it connects and a change plan.
- `proposal-reviewer` reads one proposal closely and returns findings with file and line references, plus the identifiers that pin a merge to the result it reviewed.

### Hooks

None. Oz IDE keeps history, so a replaced or deleted file remains reachable from earlier commits.

## What the plugin connects to and sends

- **MCP server.** The plugin connects to one remote server at `https://oz-ide.mcp.opzero.sh/mcp`. You sign in with an OpZero account using OAuth. No API key is stored in the plugin.
- **What is sent.** Oz IDE receives the repository ids, paths, file contents and edits that Claude sends through its tools, and stores the files you ask it to store with their history. An edit or merge can change a shared branch.
- **Large files.** A file over 1 MB cannot be sent through a tool call. When you ask for one to be stored, the server returns a short lived upload address in OpZero's storage and Claude sends the file there with an HTTP request. The `import-and-export` skill tells Claude to say which files it will upload before it does. Nothing is uploaded unless you ask to store such a file.
- **What is not sent.** The plugin has no telemetry and runs no scripts of its own.

### Optional files for other clients

These files are not run by Claude. They exist so the same product can be used from other assistants.

- `scripts/build_openai.py` builds a ZIP package for ChatGPT and Codex from the files in this repository. It reads local files and writes ZIP files to the folder you name. It makes no network calls.
- `integrations/grok_example.py` is a sample for the xAI API. It runs only when you run it. With `--dry-run` it prints a request and sends nothing. Without it, the script reads `XAI_API_KEY` from your environment and sends one request to `https://api.x.ai/v1/responses`. If `OZ_IDE_ACCESS_TOKEN` is set, that OpZero access token is included in the request so that xAI can call the Oz IDE server for you.
- `integrations/grok-servers.json`, `integrations/muse-settings.json` and the files under `integrations/openai` are configuration for those clients. They contain the same server URL and no credentials.

## Limits

- Up to 25 repositories per account. A repository id is lowercase letters, digits and hyphens, at most 23 characters.
- Up to 1 MB per file inline, and up to 25 MB per file through an upload.
- Oz IDE stores and merges text. It does not build or run code.

[Product documentation](https://oz-ide.mcp.opzero.sh/) · [Privacy](https://opzero.sh/privacy) · [Support](https://opzero.sh/support)

## Install in Claude Code

```text
/plugin marketplace add OpZero-sh/plugins
/plugin install oz-ide@opzero
```

After installing, open `/mcp` and sign in to **oz-ide**.
