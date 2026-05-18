---
name: ish
description: "Use this skill whenever the user mentions ish, a study, a tester profile, a simulation run, an \"ask\", an audience, wants to dispatch tests against AI testers, or wants to rehearse a conversation between two AI personas (e.g. sales rep vs. skeptical buyer, founder vs. investor archetype). Wraps the `ish` CLI for managing studies, asks, iterations, tester profiles, chatbot endpoints, and simulation runs against the Ish platform. Always start by running `ish docs overview` to load the domain model, then `ish docs list` and `ish docs get-page <slug>` for specifics. Prefer this skill over guessing flags from `ish --help`."
license: SEE LICENSE IN LICENSE
metadata:
  author: ish
  version: "0.16.0"
allowed-tools: Bash(ish:*)
---
# ish

A CLI for the Ish platform — run user-research studies and quick "ask"
reactions against AI tester audiences. The CLI is the agent surface;
this skill teaches you how to use it without re-reading its docs every
time.

## When to invoke this skill

The user mentioned any of: `ish`, a study, a tester profile,
a tester source, a simulation run, an iteration, an "ask", an audience,
or wants to dispatch tests against AI testers. Also invoke if the user
asks to "run a study", "generate testers", "compare variants", "test a
prototype with users", or similar.

## First step, every time: load the mental model

Before producing any `ish` command, run:

```bash
ish docs overview
```

This prints a one-page mental model (workspace → study | ask → testers
→ results) and lists every concept page available offline. The model is
non-obvious — *do not* skip this step the first time the user asks for
anything ish-related in a session.

If you need detail on a specific concept:

```bash
ish docs list                          # every page available
ish docs get-page concepts/study       # one page, full markdown
ish docs get-page concepts/run-verbs   # study run vs ask run
ish docs search "<keyword>"            # ranked hits with snippets
```

The pages `ish docs` exposes are the source of truth — newer than this
skill file. **Trust `ish docs` over anything in this skill if they
conflict.**

## Quick orientation (one-screen)

```
Workspace (= product)
├── Tester Profiles (tp-…)   reusable audience personas
│     └── Sources (tps-…)    transcripts/audio/images that seed generation
├── Study (s-…)              persistent research artifact
│     ├── modality           interactive | text | video | audio | image | document | chat
│     │                       chat has two modes: external_chatbot (probe a customer bot)
│     │                       and tester_pair (two AI personas converse — rehearsal)
│     ├── assignments        tasks the tester does
│     ├── questionnaire      questions the tester answers
│     └── Iterations (i-…)   one configured run; carries the URL or media
│           └── Testers (t-…) instance of a profile in this iteration
└── Ask (a-…)                lightweight reaction artifact
      └── Rounds             unit of execution; audience fixed at ask creation
```

Two run verbs:
- `ish study run` — dispatches simulations on the latest iteration of a study.
- `ish ask run`  — appends a round to an ask (or `--new` to create one).

Use **study** when the tester must *do* something on a real surface;
use **ask** for quick reactions to text/image variants.

**Cold-start caveat — "create a fresh workspace" is conditional on
quota headroom.** `workspace_create` returns
`error_code: usage_limit_reached` the instant the account is at
`maxProducts` (FREE caps at 1). Always inspect with `workspace_get`
first and check the `has_headroom` flag per row, or use
`ish workspace create --name <name> --ensure` — idempotent: returns
the existing workspace by name when one exists, otherwise creates. See
`ish docs get-page guides/cold-start` before producing a
workspace_create call on a session you haven't already probed.

## High-frequency commands

```bash
# First command on a cold start — confirms login + active context:
ish status              # or: ish whoami
# → user, active workspace/study/ask, token validity, API url

# Auth & active selection (saved to ~/.ish/config.json)
ish login
ish workspace use w-6ec
ish study use s-b2c
ish ask use a-6ec

# Idempotent workspace create — returns existing if name matches.
# Use this on cold-start instead of a blind workspace_create that may
# hit usage_limit_reached. See `ish docs get-page guides/cold-start`.
ish workspace create --name "Acme — onboarding" --ensure

# Inspect
ish workspace list
ish study list
ish iteration list --study s-b2c
ish ask list

# Define / configure (one-shot — iteration A inline)
ish study create --modality interactive --name "..." --url https://example.com   --assignment "..." --question "..."
ish study create --modality image --name "..."   --image-urls "https://cdn.example.com/a.png,https://cdn.example.com/b.png"   --assignment "Compare:Which feels more premium?"
ish study create --modality video --name "..."   --content-url https://cdn.example.com/ad.mp4 --assignment "Watch:..."

# Or 2-step (when you want to A/B iterations later, or upload local files)
ish study create --name "..." --modality interactive --assignment "..."
ish iteration create --url https://example.com  # auto-uploads local files

ish profile generate --description "..." --count 5

# Chat modality (external_chatbot — talk to a customer chatbot).
# Audience size lives on study run; study create defines the persistent shape only.
ish chat endpoint init --from-curl ./bot.curl --name my-bot
ish chat endpoint test my-bot -m "Hello"
ish study create --modality chat --endpoint my-bot --assignment "Sign up:Try to sign up"
# (then) ish study run --sample 5 --wait

# Chat modality (tester_pair — rehearse a conversation between two AI personas).
# Audiences are pinned to the iteration; study run refuses run-time audience
# overrides. Each side accepts EITHER explicit profiles OR a role-criteria
# filter (or both — criteria validates the explicit list).
ish study create --modality chat --chat-mode tester_pair --name "Pitch rehearsal" \
    --audience-a tp-sales-1,tp-sales-2 --audience-b tp-cto-skeptic-1,tp-cto-skeptic-2 \
    --scenario-a @./sales_rep.md --scenario-b @./skeptical_cto.md \
    --assignment "Pitch:Try to win the meeting"
# (then) ish study run -y

# Criteria-driven variant — backend resolves the eligible pool per side.
# Persona-first: the persona is sacred, criteria filter who plays the role.
ish study create --modality chat --chat-mode tester_pair --name "Pitch rehearsal" \
    --role-criteria-a '{"occupation":["sales"],"min_age":28}' \
    --role-criteria-b '{"occupation":["cto","vp engineering"],"country":["US","SE"]}' \
    --scenario-a @./sales_rep.md --scenario-b @./skeptical_cto.md \
    --assignment "Pitch:Try to land a pilot"

# Run
ish study run --sample 5 --country SE --wait
ish ask run --new --name "..." --prompt "..." --variant text:"A" --variant text:"B" --sample 30 --wants-pick --wait

# Stage an ask for human review, then dispatch (no credits charged on stage)
ish ask create --name "..." --prompt "..." --variant text:"A" --variant text:"B"     --sample 30 --wants-pick --no-dispatch
ish ask dispatch a-6ec --wait

# Results
ish study results
ish ask results a-6ec --round 1

# AI summary + key insights (any modality with completed testers)
ish study analyze --wait                                       # trigger + block
ish study insights                                             # read latest

# Screenshots (interactive studies — see what testers actually saw)
ish study screenshots                                          # list, frame-grouped
ish study screenshots download <study-id> --id <scid> --out shot.png
ish study screenshots download <study-id> --all --out ./shots/

# Chat configurations (model + system prompt + tools per chatbot endpoint)
ish chat config list                                           # active endpoint
ish chat config set --name v1 --model claude-sonnet-4-6 \
    --system-prompt-file ./prompt.txt --default
ish chat config get cc-abc --view iterations                   # cross-study use

# Read offline docs
ish docs overview
ish docs get-page <slug>
ish docs search <query>
```

## Common workflows (worked examples)

See `references/workflows.md` in this skill for end-to-end transcripts:
- First study from zero (auth → workspace → audience → study → iteration → run → results)
- Quick A/B ask with image variants
- Generating profiles from a transcript or audio source
- Targeting a gated URL (basic auth, session cookie, login form)
- Re-running a study with a fresh audience
- Extending a tester past its step cap (or redirecting mid-run with `study extend`)

## Display vs. capture: the right output mode

Three output modes — pick the one matching your intent, **don't reach
for `jq` / `python` reflexively**:

| Intent                                          | Use                  |
|-------------------------------------------------|----------------------|
| Show the user a list/table                      | bare command (TTY) or `--human` |
| Capture one value to feed into the next command | `--get <field>`     |
| Parse multiple fields / nested shape            | `--json`            |

`--get` extracts a single field from the JSON response and prints its
bare value. It supports dotted paths and auto-descends into list
`items` so `--get alias` on a paginated list yields one alias per
line. `--human` forces human output even when stdout is piped — use
it when you want to `tee` a table to a file but still show it. The
two flags are mutually exclusive (capture and display are different
intents).

### Worked example — capture in a script, display to the user

```bash
# DON'T: shim around the CLI with jq just to grab one value.
# ASK=$(ish ask create … --json | jq -r .alias)

# DO: capture mode — bare value, exit 0.
ASK=$(ish ask create --new --name demo \
        --prompt "Which?" --variant text:A --variant text:B \
        --sample 30 --get alias)

# DON'T: pipe --json through jq when you want to show the user a table.
# ish ask results "$ASK" --json | jq … | tee /tmp/x.txt

# DO: --human keeps the table layout even through tee.
ish ask results "$ASK" --human | tee /tmp/transcript.txt
```

Missing field on `--get` → exit 2 with a usage error. `--get` also
implies `--quiet` so the bare value is the only thing on stdout.

## Output handling

- Every command supports `--json`. JSON mode is **auto-enabled when
  stdout is piped**, so an agent rarely needs `--json` explicitly.
- **`--get <field>` is the right way to capture a single value.**
  Dotted paths supported (`tester_profile.name`); on a paginated
  `{items: [...]}` response, a leading non-`items` segment
  auto-descends into items. Replaces the
  `--json | jq -r .field` shim. Implies `--json` and `--quiet`.
- **`--human` forces human output even when stdout is piped.** Use it
  to `tee` a table without losing the layout. Mutually exclusive
  with `--get`.
- `--fields a,b,c` strips JSON output to the listed fields (saves
  tokens). `--verbose` adds full UUIDs and timestamps.
- **Stdout is data only.** All progress, status, and "Open in browser"
  hints go to stderr; `--json | jq -e .` parses cleanly without
  defensive piping.
- **List responses are a six-key envelope:** `{items, total, returned,
  limit, offset, has_more}`. Use `has_more` to detect truncation;
  don't count items yourself.
- **`study` JSON includes a `url` field.** `study create / generate /
  get / list / run` each return a top-level `url` (per item on
  `list`) pointing to the study in the web app — `overview` for
  read/write commands, `timeline` for `study run`. Surface it to
  the user instead of composing `<host>/<workspace>/<study>/...`
  yourself. Host follows the active backend (`app.ishlabs.io` on
  production, `localhost:3000` under `--dev`); override with the
  `ISH_APP_URL` env var.
- **Use `runtime_status`, not `status`, on study responses.** Values:
  `draft | running | completed | completed_with_errors | cancelled`.
  Derived from iteration testers' actual state — never reports
  `failed` while completed runs exist. The CLI also surfaces
  `status_inferred` + a stderr warning when raw `status` and the
  testers disagree.
- **`study generate --json` returns `modality_rationale`** (one
  short sentence). Inspect it before adding iterations; if the LLM
  picked the wrong modality, override via
  `ish study update <id> --modality text`.
- **Failed testers expose `error_message`.** `study tester --json`
  and `study results --json` (in `testers[]`) include
  `error_message: "<reason>"` for any tester with `status: failed`.
  Don't drill into logs — read the field. `study results` also
  includes a top-level `failed_count` alongside `completed_count`.
- **`ask add-questions` is additive by default.** Appending a
  follow-up question to a completed round preserves prior comments,
  picks, and ratings; only the new question is dispatched. Pass
  `--redispatch-all` for the legacy reset-and-rerun behavior.
- **`ask create --no-dispatch` stages a draft, no bill yet.** Pair
  with `ish ask dispatch <id>` to flip DRAFT → RUNNING and start
  the round. Use this when the user wants to review the audience or
  prompt before any credits are charged. Audience flags are still
  required (testers materialize at create time); only the worker
  enqueue and billing are deferred. Asks now carry a top-level
  `status` (`draft | running | completed | cancelled`) visible in
  `ask list` and `ask get`. `dispatch` is idempotent — a
  non-DRAFT ask returns 409 mapped to a usage error.
- **`ask results --json` adds `cross_round_summary` for 2+ rounds.**
  Top-level field with per-round picks/winner snapshots and
  `picks_delta` (R1 → last). Don't diff two `ask results` calls by
  hand.
- **`ask retry <ask> --round N` re-dispatches errored responses.**
  Use after a partial failure (e.g. 4 of 5 testers errored on round
  1). Only ERRORED rows are reset to PENDING and re-run; COMPLETED
  rows are left untouched. Idempotent: zero-errored is a no-op. Add
  `--wait` to block.
- **Errored ask responses carry `error_message` + `error_kind`.**
  Each `responses[]` entry whose `status: errored` exposes the
  classified failure (e.g. `first_impression_llm_failed`,
  `interview_llm_failed`, `variant_preparation_failed`). Branch on
  `error_kind` to decide retry vs abort.
- **`winner` carries `n` and `confidence`.** `n` is the completed
  sample the verdict was elected from; `confidence` is `low` /
  `medium` / `high` based on completion ratio + tied-ness. When
  errored responses exceed 50%, the winner block is REPLACED by
  `{ refused: true, reason: "error_rate_too_high", errored, total }`
  — run `ask retry` first.
- **`--workspace` works at the program root AND every subcommand.**
  `ish --workspace w-6ec study list` and `ish study list --workspace
  w-6ec` are equivalent; if both are passed, the subcommand-level
  flag wins. Without either, the CLI falls back to `ISH_WORKSPACE`
  env then the active workspace in `~/.ish/config.json`.
- **`profile generate` emits stderr progress.** `generating N
  profiles…` then `generated N profiles` around the ~10–20s LLM
  call. Suppress with `--quiet`. Generated bios reference the
  brief's domain context naturally (occupation, daily work,
  frustrations) — they no longer parrot vocabulary from the brief
  verbatim. DOBs spread across the year instead of all-on-`06-15`.
- **Empty-pool errors include a country-suggestion line.** When
  `study run` / `ask run --new` rejects because `--country XX`
  matched zero profiles, the error includes the top-3 populated
  countries that satisfy your *other* filters. Pivot directly without
  a second `profile list` round-trip.
- **`<entity> list` emits a stderr pagination hint** when
  `has_more=true` and `--quiet` is unset. Goes to stderr in **every
  mode** (including `--json` and piped stdout) — it never pollutes
  machine-readable stdout but is visible to any agent reading stderr.
  Format: "showing N–M of TOTAL; pass --offset M --limit N for more."
- **`study delete` requires explicit confirmation.** Interactive:
  prompts on stderr. Non-interactive (`--json`, piped, non-TTY
  stdin): pass `-y` / `--yes` to confirm. Without it, the CLI
  exits with usage code 2.
- **`ask add-questions` supports `--wait` / `--timeout`.** Match
  the parity of `ask create` and `ask run`. Without `--wait` the
  command returns after dispatch (round still running).
- **`study extend <tester>` resumes a terminal tester.** Use it when
  a run hit `--max-interactions` before finishing, or pair with
  `study cancel` to redirect mid-run via `--instruction` (inline,
  `@path`, or stdin via `-`). Spawns a **new** tester branched from
  the source's last interaction — source row untouched. Credits debit
  per `max(1, round(additional_steps / 10))`. See workflow #11 and
  `ish docs get-page concepts/extending-a-simulation`.
- **`pick_confidence` (0..1) is on every `--wants-pick` response.**
  The model's self-reported confidence in its variant choice. Use it
  to break ties when nominal pick counts are close. See
  `ish docs get-page concepts/ask`.
- Exit codes carry meaning: 0 success, 2 usage/validation,
  3 auth, 4 not-found, 5 transient. See
  `ish docs get-page reference/json-mode`.
- **Tier limits surface as `error_code: "usage_limit_reached"`**
  (HTTP 403, exit 1, non-retryable). The error body includes
  `tier`, `limit`, `current`, `max`, `upgrade_url`. Do not
  retry — branch on the code and surface the upgrade link. See
  `ish docs get-page reference/billing-limits`.
- Aliases (`s-…`, `a-…`, `tp-…`, `i-…`, `t-…`, `tps-…`, `w-…`)
  are accepted anywhere a UUID is. See
  `ish docs get-page reference/aliases`.

## Credits & cost preview

Every dispatched run costs **credits**. The CLI surfaces an upper-bound
estimate *before* you dispatch so you can budget:

- **Human output** — `study run` shows a `Scale:` + `Credits (est):`
  line in the confirmation block (skipped under `--yes` or `--json`).
- **JSON output** — `study run --json` includes a `credit_estimate`
  field. For tester-pair chat it nests under `pair_preview`; for
  solo/media runs it's top-level. Shape:
  `{ upper_bound: number, formula: "media_per_tester" | "chat_solo" |
  "chat_pair" | "ask_per_response", breakdown: string, unit: "credits" }`.
- **`formula` is stable** — agents can branch on it.

Today every modality uses `max(1, round(N / 10))` per principal
(per tester for media/interactive, per side per conversation for chat,
×2 for tester-pair). Asks bill flat **1 credit per successful response**.
Insights cost **10 credits flat** (first per-study is free).

If you exceed the available budget at dispatch time, the backend rejects
with HTTP 402 / `error_code: "insufficient_credits"`. The envelope
carries `required`, `available`, `upgrade_url`. Don't retry — surface
the upgrade link.

The full table (per-modality rates, tier allotments, error envelope)
lives in `ish docs get-page reference/credits`.

## Common pitfalls (don't do these)

1. **Don't paste flags from memory.** The CLI evolves; flags change.
   Run `ish <command> --help` to confirm before constructing a command.
2. **Don't pipe `--json` through `python`/`jq` to reshape output** —
   the CLI already has the affordances:
   - Inspect a few specific entities? `ish profile get tp-1b9 tp-fc1
     tp-2fc` (also works for `study get`, `iteration get`, `ask
     get`). Returns a `{items:[...], total:N}` envelope.
   - Want only certain fields? `--fields alias,name,country,occupation`.
   - Need counts of a nested array? `ask get` / `ask create --wait`
     already include `testers_count`, `responses_total`,
     `responses_complete` (per-round and aggregate). Don't recount.
   - Want machine-readable A/B verdicts? `ask results --json` already
     ships `aggregates: { picks, ratings, winner }` per round.
3. **Don't run `ish study run` against an empty study.** `ish study
   create` and `ish study generate` no longer auto-create iteration
   A — the first explicit `ish iteration create` becomes A. Running
   `study run` on a study with zero iterations exits 2; create one
   first via `ish iteration create --url …` / `--content-url …` /
   `--content-text …`. Or pass `--content-text` / `--url` directly
   on `study create` for a one-shot study + iteration A.
4. **Don't pass `--profile` together with demographic filters** — they
   are mutually exclusive. Either explicit IDs or
   `--country`/`--gender`/`--min-age`/`--max-age` + `--sample`.
5. **Don't change audience between rounds of an ask.** It's fixed at
   ask creation. Use `ish ask add-testers` to *extend* it; you can't
   replace it.
6. **Don't try to put credentials in the URL** for gated study URLs.
   Configure them once on the workspace via
   `ish workspace site-access …` (basic-auth, cookie, login).
   See `ish docs get-page concepts/site-access`.
7. **Don't commit `~/.ish/config.json`** — it stores tokens and active
   workspace/study/ask selections. It lives in `$HOME`, not the repo.
8. **Don't pass run-time audience flags to a tester_pair chat iteration.**
   Pair iterations carry their own audiences (`audience_a` /
   `audience_b` inside `details.mode_details`); `ish study run`
   refuses `--profile` / `--sample` / `--all` / demographic filters
   on them. To change audiences, update the iteration via
   `ish iteration update <id> --details-json '{...}'`. When both sides
   ship explicit `--audience-a` / `--audience-b` lists, lengths must
   match (1:1 by index) — or use `--role-criteria-a/-b` and let the
   backend resolve a pool.
9. **Don't cram demographic constraints into `scenario_a/_b` text.**
   Demographics (occupation, age, country, gender) belong in
   `--role-criteria-a/-b` so the persona stays sacred — filtering
   happens upstream of the prompt. Scenario text is for voice, goal,
   and knowledge of the role, not for who plays it. Mixing the two
   breaks the asymmetry contract and produces incoherent characters.
10. **Don't retry `usage_limit_reached` errors.** Tier caps
    (`maxProducts`, `maxStudiesPerProduct`, `maxIterationsPerStudy`,
    `maxCustomTesterProfiles`) are enforced server-side. The error body
    carries `tier`, `limit`, `current`, `max`, `upgrade_url` — show
    the upgrade link or delete an existing resource to free headroom.
    See `ish docs get-page reference/billing-limits` for the table.
11. **Don't retry `insufficient_credits` errors either.** HTTP 402,
    non-retryable. Read the `credit_estimate` field on `study run --json`
    *before* dispatching to know what you'll spend; if the error fires
    after, surface `required` / `available` / `upgrade_url` to the
    human. See `ish docs get-page reference/credits`.
12. **Don't dispatch interactive/media runs without thinking about
    `--max-interactions`.** `ish study run` defaults to a 20-step
    cap (flag > iteration's stored value > 20), which is the right
    answer for most onboarding/landing-page probes. Raise it
    (`--max-interactions 50`) when testers genuinely need to roam
    further; lower it (`--max-interactions 5`) for a smoke probe
    against a surface you suspect is broken — a stuck tester on a
    non-responsive page will otherwise burn the full cap before the
    SDK gives up. The confirmation block prints the resolved value
    and where it came from. Credits debit per
    `max(1, round(steps/10))` per tester; see
    `ish docs get-page reference/credits`.
13. **Don't call `workspace_create` blind on a cold start.** On a
    saturated account it returns `error_code: usage_limit_reached`
    immediately — the dogfood account hits this on the first call.
    Always call `workspace_get` (or `ish workspace list --json`)
    first and inspect `has_headroom` per row; if any existing
    workspace fits the work, use it via `ish workspace use <id>`.
    To programmatically reuse-or-create idempotently, prefer
    `ish workspace create --name <name> --ensure` — returns the existing
    workspace owned by the caller when the name matches, otherwise
    creates a fresh one. Same response shape either way, so the
    agent doesn't branch on success vs. reuse. See
    `ish docs get-page guides/cold-start`.
14. **Don't trust `occupation` filters as whole-token matches.**
    `audience_build` treats `occupation` as a **loose,
    case-insensitive substring** — `occupation=["manager"]` matches
    hotel managers, retail managers, bank branch managers, not just
    the engineering managers you probably wanted. Two recovery
    paths: enumerate the role surface explicitly
    (`occupation=["engineering manager", "software engineering
    manager", "vp engineering", "tech lead"]`) or read
    `match_preview` on the `audience_build` response and iterate
    on the filter before `ask_run` / `study_run`. The public
    profile pool skews non-tech / non-Western, so even a precise
    filter may resolve to a small count — preview before dispatching
    a run that depends on reaching N matches. See
    `ish docs get-page concepts/audience`.

## Authentication

`ish login` opens a browser and saves tokens to `~/.ish/config.json`.
The CLI also accepts `--token <token>` or `ISH_TOKEN` env var. If a
command exits with code 3 ("auth"), tell the user to re-run `ish login`.

## When ish is the wrong tool

If the user wants to *write code* against the Ish API directly, point
them at the API docs at https://ishlabs.io — this CLI is for
orchestration, not as an API client library.

---

**Skill version:** 0.16.0
**Skill source of truth:** `ish docs` (offline, ships with the binary)
