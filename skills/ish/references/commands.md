# ish commands — quick index

This file is intentionally thin. The authoritative, always-current
reference is `ish docs` (built into the binary) and per-command
`--help`. Use this index only for orientation — when in doubt, run:

```bash
ish docs list
ish <command> --help
```

## Top-level groups

| Group       | Purpose                                         | Concept page                |
|-------------|-------------------------------------------------|-----------------------------|
| `workspace` | Top-level container (= product). `info` shows usage caps. | concepts/workspace |
| `study`     | Persistent research artifact                    | concepts/study              |
| `iteration` | One configured run of a study (URL or media)    | concepts/iteration          |
| `ask`       | Lightweight reaction artifact                   | concepts/ask                |
| `person`    | People, people generation, and the `suggest-scenarios` + `evidence add`/`list` probe loop for crafting one specific person | concepts/person             |
| `source`    | Upload sources for person generation           | concepts/source             |
| `config`    | Simulation configs (model, timing, retries)     | (run `ish config --help`)   |
| `chat`      | Chat endpoint CRUD + smoke test (external_chatbot mode); pair-mode iterations created via `iteration create --chat-mode participant_pair` | guides/chat                 |
| `secret`    | Per-workspace secrets (`{{secret:KEY}}` resolver); `set` and `delete` block reserved `BASIC_AUTH_*`/`SESSION_COOKIE_*`/`LOGIN_*` keys | concepts/secret           |
| `check`     | Local-toolchain preflight for `study run --local` (`check ios` / `check android` / `check web`). Exits non-zero on a blocking gap so it gates: `ish check ios \|\| ish setup`. Alias: `doctor`. | guides/native-app |
| `setup`     | Install the missing local-sim deps surfaced by `ish check` (Xcode toolchain, simulators, adb, Chromium). | guides/native-app |
| `docs`      | Offline docs for agents                         | (run `ish docs --help`)     |
| `commands`  | Entire command tree + flags + exit codes as one JSON manifest (`ish commands --json`) | reference/commands |
| `init`      | Drop this skill into a Claude Code / Codex /    | (run `ish init --help`)     |
|             | Cursor / Cline / Roo project                    |                             |
| `mcp`       | Wire the hosted ish MCP server into local AI    | guides/mcp-add              |
|             | clients (Cursor, VS Code, Claude Code,          |                             |
|             | Claude Desktop, Windsurf). Idempotent.          |                             |
| `login`     | Browser-based auth — **always runs the flow** (re-auths every time; not a status check — use `status` to *check* sign-in). Each login mints a fresh OAuth client; for automation use `--token`/`--token-file`/`ISH_TOKEN`. | concepts/active-context     |
| `logout`    | Clear saved credentials AND active context (workspace/study/ask/chat_endpoint) — it's an identity switch | concepts/active-context     |
| `status`    | Show active session (user, workspace,           | concepts/active-context     |
|             | study, ask, signed-in state) — alias `whoami`   |                             |
| `connect`   | Cloudflare tunnel exposing localhost so Ish's REMOTE fleet can reach it. For a local WEB app, prefer `study run --local` (browser on your machine, no tunnel); use `connect` only for the remote fleet. | concepts/run-verbs |
| `upgrade`   | Self-update                                     | —                           |
| `feedback`  | Report a bug / feature request / note to the ish team. `--health` attaches setup checks + local-sim logs. | guides/feedback |

## Discovering flags safely

Don't construct `ish` commands from memory for anything beyond the
high-frequency examples. Run:

```bash
ish <group> --help              # group-level summary + concept primer
ish <group> <verb> --help       # exact flags for the verb
```

Every group's `--help` ends with a "Concept pages" footer pointing at
the right `ish docs get-page <slug>` to read deep context.

## Aliases

Short prefixed IDs (e.g. `s-b2c`, `p-795`, `a-6ec`, `i-d4e`,
`pt-a17`, `ps-3a4`, `w-6ec`, `c-c3c`) are accepted anywhere a UUID
is expected. Full UUIDs always work too. See
`ish docs get-page reference/aliases`.

## Output flags (global)

| Flag             | Effect                                                   |
|------------------|----------------------------------------------------------|
| `--get <field>`  | **Capture mode.** Print the bare value at the dotted path; accepts bracket indexing too (`"items[0].alias"` == `items.0.alias`); auto-descends into list `items`. Implies `--json` + `--quiet`. Missing field → exit 2 (`validation_error`). Replaces `--json \| jq -r .field`. |
| `--human`        | **Force display mode** even when stdout is piped (overrides JSON-when-piped). Mutually exclusive with `--get`. |
| `--json`         | JSON output (auto-on when stdout is piped)               |
| `--fields a,b`   | Keep only listed fields in JSON                          |
| `--list-fields`  | Print the projectable field names (`{"fields":[...]}`) instead of the data — discover the shape without guessing. `--fields ?` is shorthand. |
| `--verbose`      | Include UUIDs + timestamps in JSON                       |
| `-q, --quiet`    | Suppress progress messages on stderr                     |
| `-t, --token`    | Auth token (else ISH_TOKEN env, else `ish login` saved)  |

See `ish docs get-page reference/json-mode` for the full display-vs-
capture-vs-chain decision rule.

## Exit codes

`0` ok · `1` general (incl. `usage_limit_reached` /
`insufficient_credits` billing walls — NOT exit 3) · `2`
usage/validation (incl. `ConfirmationRequired` from the credit-spend
gate and client-side input checks) · `3` auth · `4` not-found ·
`5` transient (retryable). See
`ish docs get-page reference/json-mode`.
