---
name: ish
description: "Use this skill whenever the user mentions ish, a study, a person, a simulation run, an \"ask\", a group of people, a chatbot probe, wants to dispatch tests against AI participants, or wants to rehearse a conversation between two AI personas (e.g. sales rep vs. skeptical buyer). Covers both the `ish` CLI (via Bash) and the hosted ish MCP server (`mcp__claude_ai_ish__*` on claude.ai) — same operations, pick whichever your environment has. Read this skill first to orient on the mental model, then trust `ish docs` (CLI) or the MCP tool descriptions for argument details."
license: SEE LICENSE IN LICENSE
metadata:
  author: ish
  version: "0.27.0"
---
# ish

ish runs user-research simulations: simulated people experience your draft (page, copy, ad, pitch, chatbot, video, document) and report what they noticed, where they stalled, what they would do next. Use before shipping, when you need a fast reaction round, or to rehearse a conversation between two AI personas.

## When to invoke

The user mentioned `ish`, a study, an "ask", a person, a group of people, a simulation, "rehearse", "compare variants", "test before shipping", "probe a chatbot".

## Drivers

ish has two surfaces; pick whichever your environment has:

- **MCP** — `mcp__claude_ai_ish__*` on claude.ai. Tool descriptions are authoritative for argument schemas.
- **CLI** — the `ish` binary. `ish --help` per command; `ish docs overview` / `ish docs list` / `ish docs search` / `ish docs get-page <slug>` for concept docs.

Both wrap the same operations. If neither is present, tell the user: `npm i -g @ishlabs/cli`, or enable the ish connector on claude.ai. Don't try to drive ish without a driver.

**Bridging CLI → MCP for the user's editor / desktop agent**: if the user has the CLI but their editor or desktop agent (Cursor, VS Code, Claude Code, Claude Desktop, Windsurf) isn't yet wired to call ish, one command does it: `ish mcp add --all --yes`. Writes the per-client MCP config block, never embeds a token (OAuth on first connect), idempotent. See `ish docs get-page guides/mcp-add`.

**When both are available, pick by op:**
- Streaming results to a watching user → **CLI** with `--wait` (per-participant output as participants complete).
- Structured one-shot reads or run dispatch → **MCP** (JSON in, JSON out, no shell).
- Idempotent setup (e.g. cold-start workspace) → **CLI** has `--ensure`; MCP doesn't.
- Local file uploads (images, video, docs) → **CLI** only — MCP doesn't accept binaries.

**Naming convention in this skill**: shapes below use MCP tool names (`ask_run`, `study_create`, `chat_endpoint_init`, …). The CLI equivalents are the same names kebab-cased under a noun group (`ish ask run`, `ish study create`, `ish chat endpoint init`, …). When in doubt: `ish --help` or `ish <noun> --help`.

## Mental model

```
Workspace (= product)
├── Person (p-…)    reusable AI persona
├── Study (s-…)              persistent artifact for testing a real surface
│   └── Iteration (i-…)      one configured run; carries the URL or media
├── Ask (a-…)                lightweight artifact for reactions to text/image variants
│   └── Round                unit of execution; participants fixed at ask creation
└── Chat Endpoint            workspace-level definition of an external chatbot
                              (referenced by study modality: chat, mode: external_chatbot)
```

**Audience is a query, not an entity.** Both `ask_run` and `study_run` take an `audience` argument shaped as `{ person_ids: [...] }` (explicit) or `{ sample: N, filters: {...} }` (sampled from an existing pool). There is no `audience` resource to create — you build profiles via `group_build` (or reuse existing ones via `profile_list`) and pass them in.

Two run verbs:
- **study run** — simulate on a real surface (URL, media, document, chat endpoint).
- **ask run** — react to text or image variants.

Heuristic: **study** for "test this prototype/page/flow"; **ask** for "which copy/image lands better".

## Workflow shapes

Each shape names the verb, the *required precursors*, and the **load-bearing knobs** — the arguments that change output quality, not just behavior. Look up the full schema in the MCP tool description or `ish <command> --help` once you've picked the shape.

Examples below use MCP shape; for CLI, kebab-case the tool name (`ask_run` → `ish ask run`) and pass equivalent flags (`person_ids: [...]` → `--person-id p-… --person-id p-…`).

### Compare text or image variants → `ask_run`

- **Precursor**: a group of people (see "Audience is a query" above). If you don't already have suitable people, build them first via `group_build`; reuse via `profile_list` when possible.
- **Load-bearing knobs**:
  - `wants_pick: true` — adds an aggregate winner verdict. Without it you get prose reactions but no clear answer.
  - `wants_ratings: true` — adds per-variant numeric scores.
  - `wait: true` — block until done. Without it you get a round id and have to poll.
  - `variants` — array of `{ label, content }` for text, or `{ label, image_url }` for hosted images. Two or more variants required for `wants_pick` to be meaningful (with N=1 it degrades to a prose reaction round). **Local image files**: only the CLI accepts them. Use `--variant LABEL:@./path.png` per file (the `@` prefix triggers upload); MCP requires a hosted URL.
  - `ask_id` (optional) — passing an existing `a-…` id re-runs against that ask. Omit (or pass `--new` on the CLI) to create a new ask in one shot.
- **Shape**:
  ```
  ask_run({
    variants:  [ { label: "A", content: "..." }, { label: "B", content: "..." } ],
    audience:  { person_ids: ["p-…", ...] },   // or { sample: 10 }
    wants_pick: true,
    wants_ratings: true,
    wait: true,
  })
  ```
- **Output**: per-participant reasoning + (if `wants_pick`) aggregate winner with confidence.

### Test a live page or prototype → `study_run` (modality: interactive)

- **Precursor**: a study with a URL. Either inline at create-time (`study_create({ modality: "interactive", url: "..." })`) or as a separate iteration (`iteration_create({ study_id, url })`) when you want to A/B iterations later or upload local files. An **assignment** is required — what the participant is supposed to attempt.
- **Audience**: pass `audience: { person_ids: [...] }` or `{ sample: N }` to `study_run`, same contract as `ask_run`. Audience is set on the *run*, not the study.
- **Load-bearing knobs**:
  - `assignment` (on `study_create`) — what the participant is supposed to do. Format: `"<label>:<instruction>"`. The whole run hinges on this being clear.
  - **steps (optional checklist)** — an assignment can carry an ordered `steps` list of atomic actions (`{name, description?}`), authored via the CLI JSON forms (`--assignments-file` / `--assignments`) — not the `"<label>:<instruction>"` shorthand. Honored for **interactive** and **external_chatbot chat** only. After a run, `study get` reports a per-step `step_completion` rollup (pass rate + sample failures). Use steps when "did they finish?" is a checklist, not a single yes/no.
  - `wait` (MCP) / `--wait` (CLI) — streams per-participant results as they complete. CLI streams to stdout in real-time; MCP blocks until the whole run finishes. For a watching user, prefer the CLI here.
  - `count` (on `study_run`) — how many participants.
- **Shape**:
  ```
  study_create({
    modality: "interactive",
    url: "https://staging.acme.io/welcome",
    assignment: "Complete signup:Go through the 4-step wizard end-to-end",
  })
  study_run({ study_id: "s-…", audience: { person_ids: [...] }, count: 15, wait: true })
  ```
- **Output**: per-participant journey transcripts + aggregate friction / blocker / positive-moment counts.

### Probe a customer chatbot → `study_run` (modality: chat, mode: external_chatbot)

- **Precursors**:
  1. A **chat endpoint** definition at the workspace level. `chat_endpoint_init` from a curl spec (handles auth headers, request/response shape; **upsert-by-name** — safe to re-call with the same `name` to rotate auth or change the request shape) → `chat_endpoint_test` to confirm it responds correctly before dispatching simulated participants.
  2. A study with `modality: "chat"`, `mode: "external_chatbot"`, the endpoint reference, and an `assignment`.
- **Audience**: same `{ person_ids } | { sample }` contract; pass to `study_run`. For custom personas (e.g. "frustrated vs polite"), `group_build` first.
- **Load-bearing knobs**:
  - `assignment` — what the participant tries to do (`"Cancel:Try to cancel your subscription"`).
  - `count` on the run.
- **Shape**:
  ```
  chat_endpoint_init({ name: "support-bot", from_curl: "..." })  // or describe request shape directly
  chat_endpoint_test({ endpoint: "support-bot", message: "hi" })
  study_create({ modality: "chat", mode: "external_chatbot", endpoint: "support-bot",
                 assignment: "Cancel:Try to cancel your subscription" })
  study_run({ study_id: "s-…", audience: { person_ids: [...] }, count: 8, wait: true })
  ```
- **Output**: full conversation transcripts per participant + aggregate success / blocker analysis.

### Test a media artifact (document, image, video, audio) → `study_run`

- **Precursors**:
  1. A study with the chosen modality: `study_create({ modality: "document" | "image" | "video" | "audio", assignment: "..." })`.
  2. An **iteration** carrying the media. For local files, **CLI only** — `ish iteration create --study s-… --media @./deck.pdf` (the `@` prefix triggers upload). For hosted URLs, either driver works: `iteration_create({ study_id, content_url: "https://..." })`.
- **Audience**: same `{ person_ids } | { sample }` contract; pass to `study_run`. Reusable across runs (see "Lifecycle" below).
- **Load-bearing knobs**:
  - `assignment` on `study_create` — for review-style media (decks, ad creative), frame as decision: `"Take a first meeting:Review this Series A deck and decide whether you'd take a first meeting"`. Page/timestamp-level attribution depends on the assignment asking for it explicitly.
  - `wait` / `--wait` — same streaming story as interactive.
  - `count` on `study_run`.
- **Iterating on the artifact** (v2 deck, v3 deck): create a **new iteration** on the same study (`iteration_create`), reuse the people's `person_ids`. See "Lifecycle".
- **Output**: per-participant reactions to the artifact + aggregate themes.

### Rehearse a conversation between two AI personas → `study_run` (modality: chat, mode: participant_pair)

**If the user might want the same persona across multiple turns, pin profiles up-front — you can't retro-pin after a run.** Without pinning, personas are re-synthesized from the assignment text each time, so "the same VC from earlier" becomes prose-only continuity.

- **Precursor**: a workspace and (optionally) one or two people for persona pinning. If you skip the people, ish synthesizes both personas from the `assignment` text per-run — fine for one-shot rehearsals, drifts between iterations.
- **Audience**: optional. For persona continuity across iterations, build profiles via `group_build` (or reuse via `profile_list`) and pass `audience: { person_ids: [...] }` to `study_run` — the same profiles play the same roles each time.
- **Load-bearing knobs**:
  - `assignment` — encodes BOTH personas and what each is trying to do. More prose-heavy than other assignments; be specific. Example: `"Founder pitches Series A to skeptical VC. Founder: defends AI customer-support startup, $2M ARR, 15% MoM. VC: thinks SaaS-for-SaaS is saturated, probes moat and unit economics."`
  - `count` — typically 1 per run; set higher to generate variations.
- **Iterating the scenario** (turn-by-turn refinement): create a **new iteration** with a revised assignment; reuse the same `person_ids` if you pinned personas. See "Lifecycle".
- **Output**: a full transcript per rehearsal.

### Generate a fresh group → `group_build`

- **Input**: a `description`, a `count`, and optionally `sources` (transcripts / audio / images / docs that seed persona generation — for "make profiles that feel like these real customers"). Local files force CLI (binary upload constraint).
- **Output**: a list of `person_ids` to pass into `ask_run` or `study_run`.
- **Usage**: slow (~30-120s) + draws credits. Reuse profiles via `profile_list` when possible. Sensible defaults: `count: 5-10` for ad-hoc tests, `count: 20+` for studies where you want statistical signal.
- **Growing a group of people**: build only the delta — don't rebuild. Concat the new `person_ids` with the existing ones for the next run. The "audience is a query" framing means there's no audience entity to update.
- **Shapes**:
  ```
  // Simple — description only
  group_build({
    description: "Parents of toddlers (ages 1-3), US, evening-routine focused",
    count: 8,
  })
  // → { person_ids: ["p-…", ...] }

  // Seeded from real transcripts (CLI only for local files)
  // ish person generate --description "..." --count 10 \
  //   --source @./interviews/customer-1.md \
  //   --source @./interviews/customer-2.md
  ```

## Lifecycle (what to re-use vs create anew)

The most common multi-turn question: "user wants to change X — re-use the existing thing or create a new one?"

| Change you want | What to do |
|---|---|
| Same ask, **same participants**, new variants | Pass `ask_id` (MCP) or `--ask` (CLI) on `ask_run` — re-uses the locked participants. |
| Same ask, **different participants** | New ask: omit `ask_id` (MCP) or pass `--new` (CLI). Participants are locked at ask creation. |
| Same study, **new media** (v2 deck, new image) | New **iteration** on the same study (`iteration_create({ study_id, content_url \| --media @path })`). Iterations are immutable once they have results — never edit. |
| Same study, **new assignment** | **New study.** Assignment lives on the study; there's no in-place edit. Keep the old study's id for side-by-side comparison. *(Participant-pair exception: the assignment IS the content there — use a new **iteration** on the same study, not a new study.)* |
| Same people across multiple runs / studies | Reuse the `person_ids` array. Profiles are workspace-scoped resources (`p-…`) — they live independently of any ask or study. |
| Chat endpoint definition needs to change (auth rotate, URL change) | `chat_endpoint_init` is **upsert-by-name** — re-init with the same `name` and a new `from_curl` spec. Re-run `chat_endpoint_test` to confirm. |
| Persona reuse in participant-pair | Pin via `person_ids` on the first `study_run`; pass the same ids on subsequent runs. Without pinning, personas are re-synthesized from the assignment per run. |

When in doubt: side-by-side comparison usually beats in-place edits. Ids are cheap; result history isn't.

## Sharing results (no-login link)

To hand a study to someone **without an ish account** — a prospect, a stakeholder — create a public share link. `ish study share [study]` prints a no-login `share_url` to the web viewer (summary, key insights, participant journeys, interactive frames, segment breakdowns). `ish study share --list` lists your links; `ish study unshare <token>` revokes one (takes the **raw token**, not a study id/alias). `--expires <days>` auto-expires the link. Brand the link by setting a workspace logo first: `ish workspace update <id> --logo <url>` — the logo shows on the shared page. Share **after** the study has run + been analyzed, so the viewer renders the summary + insights. Deep dive: `ish docs get-page concepts/sharing`. (CLI-only — the MCP has no share tool yet.)

## Pitfalls

- **Cold start on free plan**: `workspace_create` returns `usage_limit_reached` at the free-plan cap (1 workspace). Always inspect with `workspace_list` first. **MCP-only recipe** (no `--ensure` available): `workspace_list` → if non-empty, use the first; if empty, `workspace_create`; if `workspace_create` returns `usage_limit_reached`, re-call `workspace_list` (a workspace exists you didn't see — possibly created by another session). **CLI shortcut**: `ish workspace create --name <name> --ensure` is idempotent by name.
- **Ask participants vs variants** — see Lifecycle table for the re-use vs new-ask decision.
- **Study iterations are immutable once they have results** — see Lifecycle table for new-iteration vs new-study.
- **Credit usage**: `ask_run`, `study_run`, and `group_build` draw credits — this is the normal, expected way to use ish, so run them without hesitation. Credits are a usage allowance (paid plans refill monthly; the free tier is a one-time signup grant), not a per-call bill. Check `workspace_get`'s `credits` headroom before dispatching large runs. For free-plan ad-hoc tests, default `count: 5-8` participants + 2 variants comfortably fits the signup grant.
- **`group_build` may return fewer profiles than requested** if the description is over-constrained. Always read the returned `person_ids` count, don't trust the requested `count` blindly.
- **Variants of wildly different length** (one-line vs paragraph) can skew picks toward the longer one. Keep variants comparable in shape.
- **Chatbot endpoint response-shape mismatch**: `chat_endpoint_test` succeeds shallowly if the bot responds at all, but a wrong response path (e.g. bot returns `{ data: { reply } }` instead of `{ reply }`) produces empty transcripts on the actual run. Inspect one full test response before dispatching participants.
- **Chatbot auth drift**: tokens/sessions baked into `--from-curl` expire. If transcripts come back as identical short error strings, re-run `chat_endpoint_test` and refresh the curl spec.
- **401 surfaces as fake blocker**: an unauthenticated endpoint produces "participant got stuck on auth screen" — looks like a UX blocker but is config. Always confirm endpoint auth before reading transcripts as user-research data.
- **Don't poll a stuck run forever**: a participant whose worker died will sit in `status: running` until the backend reaper transitions it to `failed, error_kind: stale_worker` (~15 min). The per-participant status payload exposes `age_seconds` (server-computed from `started_at`); once it's above ~900s on a non-terminal row, the run is almost certainly stuck. The CLI's `wait_timeout` envelope explicitly flags this case in its `error` message — when you see "the worker likely died," stop polling and surface the failure rather than retrying. `error_kind: self_timeout` is the same idea but written by the worker itself when it self-aborts past its 25-min ceiling.
- **No per-page/per-timestamp scoping for media**: there's no "evaluate just slide 14" or "react to seconds 0-30" API. State the focus explicitly in the `assignment` text, or pre-stitch the artifact (e.g. replace one slide locally, upload as a new iteration).
- **`study get --json` participants live at the top level**, not nested under `iterations[*].participants`. The backend split made `/studies/{id}` lite (metadata + iteration shells, no participant graph) and added `/studies/{id}/participants`; the CLI joins them so `study get --json` carries a flat `participants[]` with `iteration_id` on each row. Read `.participants[]`, not `.iterations[].participants[]`.
- **All destructive deletes require `--yes` in non-TTY mode**: `ish workspace delete`, `study delete`, `ask delete`, `person delete`, `source delete`, `chat endpoint delete`. In `--json` mode (or any piped/non-TTY invocation), omitting `--yes` refuses with `error_kind: "ConfirmationRequired"` + an `example` field showing the same command with `--yes` appended. `workspace delete` is the highest-blast-radius: it removes ALL nested studies, asks, people, secrets, configs, sources, and chat endpoints — the prompt names them explicitly.
- **`ish login` is idempotent**: with a saved token that is unexpired *and* still accepted by the API, `ish login` short-circuits with "Already logged in" and **does not open a new browser tab**. If the token is unexpired but the server rejects it (revoked, rotated signing key, or minted against the wrong env — e.g. a dev-Supabase token while calling the prod api), it re-runs the browser flow instead of falsely reporting success. Use `--force` (or `-f`) only when actually switching accounts.
- **`ish person create` accepts inline flags** (mirrors `person update`): the file-only API (`--file <path>`) is preserved as an escape hatch but the common path is `ish person create --name "X" --type ai --country US ...` — `--type` defaults to `ai` when `--file` is omitted. See `ish person create --help` for the full inline-flag set including `--household` (MECE rule applies) and `--accessibility-profile`.
- **`ish status` now surfaces `chat_endpoint`** alongside `workspace`/`study`/`ask`. Stale or orphan active refs get a `warning` + `hint` field on the affected ref (instead of silently dropping the `name`). On `workspace use <other>`, the CLI cascade-clears `study`/`ask`/`chat_endpoint` (they belong to the previous workspace).
- **Share link URL host ≠ API host**: `ish study share` prints the backend-built `share_url` (the web frontend host). Use it verbatim — never reconstruct the URL from the API host or app URL; they differ. `ish study unshare` takes the **raw token** (from `study share` / `study share --list`), not a study id or alias.
- **Native app iterations (ios/android) name the app, not a URL**: `ish iteration create --platform ios --app <bundle-id>` stores the target as `app_artifact` (no URL). The iteration remembers it, so `ish study run --local` needs **no `--app` on reruns** (it defaults from the iteration). Pass `--app <path-to.app|.apk>` only to override with a fresh local build. `--app` is optional at create time (omit it for "chosen at run time"). Only `browser`/`figma` iterations require `--url`.
- **Local runs have no server-side screenshots**: `ish study run --local` (including ios/android) writes a per-step HTML debug report to `~/.ish/debug/sim-*.html` (path printed at the end of the run) instead of pushing screenshots to the server. `ish study screenshots list` on a local-only study finds none — open the debug report instead.

## When in doubt

`ish docs` (deep concept references, CLI-side) and live MCP tool descriptions (argument schemas, MCP-side) are closer to source-of-truth than this skill. **Trust them over this skill if they conflict.**

- **CLI present**: `ish docs overview`, `ish docs get-page concepts/run-verbs`, `ish docs get-page guides/cold-start`, `ish docs search <keyword>`.
- **MCP only**: read the tool description of the MCP tool you're about to call; cross-reference against this skill's "Shape" blocks. The MCP server's own `instructions` block (delivered automatically with the tool list) covers vocabulary and posture and is authoritative.
