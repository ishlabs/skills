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
| `person`    | People, people generation, and the `suggest-scenarios` + `evidence add`/`list` probe loop for crafting one specific persona | concepts/person             |
| `source`    | Upload sources for person generation           | concepts/source             |
| `config`    | Simulation configs (model, timing, retries)     | (run `ish config --help`)   |
| `chat`      | Chat endpoint CRUD + smoke test (external_chatbot mode); pair-mode iterations created via `iteration create --chat-mode participant_pair` | guides/chat                 |
| `secret`    | Per-workspace secrets (`{{secret:KEY}}` resolver) | concepts/secret           |
| `docs`      | Offline docs for agents                         | (run `ish docs --help`)     |
| `init`      | Drop this skill into a Claude Code / Codex /    | (run `ish init --help`)     |
|             | Cursor / Cline / Roo project                    |                             |
| `mcp`       | Wire the hosted ish MCP server into local AI    | guides/mcp-add              |
|             | clients (Cursor, VS Code, Claude Code,          |                             |
|             | Claude Desktop, Windsurf). Idempotent.          |                             |
| `login`     | Browser-based auth. Idempotent: short-circuits only when the saved token is unexpired AND server-accepted. `--force` to switch accounts. | —                           |
| `logout`    | Clear saved credentials                         | —                           |
| `status`    | Show active session (user, workspace,           | concepts/active-context     |
|             | study, ask, token validity) — alias `whoami`    |                             |
| `connect`   | Cloudflare tunnel exposing localhost            | —                           |
| `upgrade`   | Self-update                                     | —                           |

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
`t-a17`, `ps-3a4`, `w-6ec`, `c-c3c`) are accepted anywhere a UUID
is expected. Full UUIDs always work too. See
`ish docs get-page reference/aliases`.

## Output flags (global)

| Flag             | Effect                                                   |
|------------------|----------------------------------------------------------|
| `--get <field>`  | **Capture mode.** Print the bare value at the dotted path; auto-descends into list `items`. Implies `--json` + `--quiet`. Replaces `--json \| jq -r .field`. |
| `--human`        | **Force display mode** even when stdout is piped (overrides JSON-when-piped). Mutually exclusive with `--get`. |
| `--json`         | JSON output (auto-on when stdout is piped)               |
| `--fields a,b`   | Keep only listed fields in JSON                          |
| `--verbose`      | Include UUIDs + timestamps in JSON                       |
| `-q, --quiet`    | Suppress progress messages on stderr                     |
| `-t, --token`    | Auth token (else ISH_TOKEN env, else `ish login` saved)  |

See `ish docs get-page reference/json-mode` for the full display-vs-
capture-vs-chain decision rule.

## Exit codes

`0` ok · `1` general · `2` usage/validation · `3` auth ·
`4` not-found · `5` transient (retryable). See
`ish docs get-page reference/json-mode`.
