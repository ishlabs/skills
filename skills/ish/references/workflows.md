# ish workflows — worked examples

Each workflow below is a complete transcript an agent can adapt. Run
`ish docs overview` first if you haven't already this session.

## 1. First study from zero

Goal: from a fresh install to a finished interactive study with 3
participants and one question.

```bash
# 1. Authenticate (browser flow, saves tokens to ~/.ish/config.json)
ish login

# 2. Create + select a workspace
ish workspace create --name "Demo" --base-url https://example.com
ish workspace use w-…

# 3. Generate a small group of people
ish person generate \
    --description "Tech-savvy millennials in the US who use mobile banking" \
    --count 3

# 4. Define the study + iteration A in one call (one-shot path).
#    The same shape works for image (--image-urls), video / audio /
#    document (--content-url <url>), and chat (--endpoint <id>).
ish study create --name "Onboarding UX" --modality interactive \
    --url https://example.com --screen-format desktop \
    --assignment "Sign up:Complete the signup flow" \
    --question "How easy was it?"
ish study use s-…

# (Optional) add a B variant later instead of inline:
# ish iteration create --url https://example.com/v2

# 6. Run, blocking until done
ish study run --all --wait

# 7. Read results
ish study results --json | jq .
```

### 1a. Give the assignment a step-by-step checklist

When "did they finish?" is a checklist rather than a single yes/no, attach
`steps` to the assignment. Steps are JSON-only (no inline shorthand) and
honored for **interactive** + **external_chatbot chat** modalities only.

```bash
# assignments.json
# [
#   { "name": "Buy", "instructions": "Add an item to cart and check out",
#     "steps": [
#       { "name": "Find a product", "description": "Browse to any item" },
#       { "name": "Add to cart" },
#       { "name": "Complete checkout" }
#     ] }
# ]
ish study create --name "Checkout" --modality interactive \
    --url https://shop.example.com \
    --assignments-file ./assignments.json
ish study use s-…
ish study run --all --wait

# After the run, each step gets a pass-rate rollup:
ish study get s-…                       # human: "✓ Add to cart 4/5 (80%)" per step
ish study get s-… --json --verbose      # step_completion[] incl. sample_failures[].participant_id
```

## 2. Quick A/B ask with image variants

Goal: ship 30 simulated reactions to two hero images, with a "which do
you prefer" question.

```bash
ish ask run --new --name "hero shots" \
    --prompt "Which feels more premium?" \
    --variant image:./hero-a.png::label=A \
    --variant image:./hero-b.png::label=B \
    --sample 30 --wants-pick --wait

# Read the verdict directly — no comment-parsing required:
ish ask results --json | jq '.rounds[0].aggregates'
# → { "picks": { "A": 22, "B": 8 },
#     "winner": { "label": "A", "count": 22, "tied": false, "n": 30, "confidence": "high" } }
```

For `--wants-pick` / `--wants-ratings` rounds, `ask results --json`
adds an `aggregates` field per round with `picks`, `ratings` (mean
+ n per variant), and a `winner`. See `ish docs get-page
reference/json-mode` for the full shape.

Add a follow-up round with no participant change:

```bash
ish ask run --prompt "Which one would you click on?" \
    --variant image:./hero-a.png::label=A \
    --variant image:./hero-b.png::label=B \
    --wait
```

## 3. Generate profiles from a real source

Goal: turn a customer interview transcript into a 4-person group.

`person generate` is an async agentic job: it reads your brief and any
uploaded sources (transcripts, emails, PDFs, audio, images) describing how
real people reacted, then produces profiles PLUS scenarios grounded in those
reactions. It enqueues, polls ~30-60s, then prints the profiles (with
scenarios attached unless `--no-scenarios`). `--json` returns
`{job: {person_ids}, profiles: [...]}`.

```bash
# Inline — auto-uploads the file:
ish person generate --source ./interviews/sarah.txt --count 4

# The per-source note is the researcher's: how the person reacted to THAT file.
ish source upload ./proposal.eml --description "called this proposal lazy and vague"
# → ps-3a4 (status: processed)
ish person generate --description "Skeptical enterprise buyer" --source ps-3a4 --count 1 --json

# Or upload once and reuse the source alias:
ish source upload ./call.mp3 --diarize
# → ps-3a4 (status: processed)
ish person generate --source ps-3a4 --propose-count
# → { proposed_count: 4, rationale: "..." }
ish person generate --source ps-3a4 --count 4
```

## 4. Build a specific simulated person from notes

Goal: rebuild one named persona (a real prospect, a stakeholder for
a pitch rehearsal) via the iterative probe loop — distinct from
`person generate`, which is for groups.

```bash
# 1. Suggest 5 probes from a context blob
ish person suggest-scenarios \
    --context "Staff platform engineer at a Stripe-using fintech. \
        Owns oncall for the payments edge. Burned by a Black Friday \
        outage last year." \
    --count 5
# → {scenarios: [{type:"situation",...},{type:"binary",...},...]}

# 2. (offline) Answer the probes — build answers.json:
#    [{"text":"...","source":"situation","scenario_prompt":"..."}, ...]
#    Valid source values: situation, voice, binary, micro-story

# 3. Save the person shell — either from file:
ish person create --file ./persona.json
# → p-d4e
#
# …or inline (mirror of person update):
# ish person create --name "Alice" --type ai --country US \
#     --occupation founder --household single --bio "..."

# 4. Persist the answers as structured evidence
ish person evidence add p-d4e --traces-file ./answers.json

# 5. Read back what's saved (also useful before the next probe round)
ish person evidence list p-d4e
```

To iterate, feed prior prompts/answers back in so the LLM doesn't
paraphrase what you already asked:

```bash
ish person suggest-scenarios \
    --context-file ./notes.md --count 3 \
    --already-surfaced '["PagerDuty fires at 02:00."]' \
    --previous-answers @./answers.json
```

See `ish docs get-page guides/build-specific-person` for the full
walkthrough including the four probe-type shapes.

## 5. Target a gated URL (Vercel preview / staging gate / login form)

Configure credentials once on the workspace; participants reuse them.

```bash
# Show what's configured:
ish workspace site-access status

# HTTP basic auth:
ish workspace site-access basic-auth --username alice --password hunter2

# Session cookie (Vercel preview, Lovable, etc.):
ish workspace site-access cookie --name session --value abc123

# Login form (typed by the participant into the page):
ish workspace site-access login --username demo --password demo
```

Keep secrets out of shell history by passing `-` for `--password` /
`--value` and piping from stdin:

```bash
printf %s "$STAGING_PW" | ish workspace site-access basic-auth \
    --username alice --password -
```

## 6. Re-run a study with a fresh group

Goal: same study, same iteration, but compare groups.

```bash
# First run — Swedish 35-50:
ish study run --country SE --min-age 35 --max-age 50 --sample 5 --wait

# Second run — every female person in the workspace, same iteration:
ish study run --gender female --all --wait

# Free-text filters: --search matches the person **name**, --bio
# matches the person **bio**, --occupation matches the person
# **occupation** (repeatable, OR-joined). All are case-insensitive
# substrings — the same flag set works on `ish person list`,
# `ish ask run`, `ish ask add-people`, and `ish ask create`.
ish study run --bio "screen reader" --all --wait
ish study run --occupation founder --occupation designer --sample 6 --wait
```

If you don't pass any people flags, `ish study run` reuses the
iteration's existing participants — useful for re-running after fixing the
target page.

## 7. Localhost target (dev environment)

Expose a port via a Cloudflare tunnel; `ish connect` prints the public
URL the study iteration can point at. `connect` is foreground and
long-running — keep it open in a separate terminal (or background it).

```bash
# Terminal A — keep open:
ish connect 3000
# stdout: Tunnel URL: https://<random>.trycloudflare.com → http://localhost:3000

# Terminal B — use the URL:
ish iteration create --url https://<random>.trycloudflare.com
ish study run --sample 3 --wait
```

For agents/scripts, run `connect` in the background with `--json` and
read the URL from stdout (one JSON line per state change):

```bash
ish connect 3000 --json > /tmp/ish-tunnel.log &
sleep 3
URL=$(jq -r 'select(.status=="connected") | .tunnel_url' /tmp/ish-tunnel.log | head -1)
ish iteration create --url "$URL"
```

## 8. Chat-modality study (drive a chatbot endpoint)

The chat modality has **two modes**, picked by
`iteration.details.mode_details.mode`:

- **`external_chatbot`** — participants probe a customer chatbot endpoint
  (the original chat behaviour). Audience size is set on `study run`.
- **`participant_pair`** — two AI people converse with each
  other. Each side has its own scenario + goal; the other side does
  not see it (asymmetry contract). Audiences are pinned to the
  iteration: equal counts zip 1:1 by index, or one side of 1
  broadcasts across the other (1 × N → N conversations). Useful for rehearsing
  a sales call, a fundraising chat, a difficult conversation, or any
  two-role scenario before it happens. See section 7b below.

### 7a. external_chatbot — drive a customer chatbot endpoint

Goal: configure a customer chatbot endpoint, smoke test it, and run
a chat-modality study end to end. The CLI talks to the endpoint
through whatever transport it's configured for (sync / async-poll /
streaming); local bots reach ish via `ish connect`.

Endpoint shape is slots-only: `incoming.slots[]` lists interactive
containers (`{containerPath, kind?, labelPath?, idPath?}`,
`kind` ∈ `alternatives` / `form` / `text` or unset to
auto-classify per-turn) and `incoming.references[]` lists passive
containers (citations / followups / artifacts). Auto-detect derives
both from the response stub via shape rules — no markers to learn.

For well-known providers, skip auto-detect and start from a
hand-curated template:
`ish chat endpoint init --template <openai|anthropic|voiceflow|dialogflow-cx|botframework>`.
Templates use `{{secret:NAME}}` placeholders for auth.

```bash
# 1. Author the endpoint from a curl example (or a ChatbotEndpointConfig file).
#    Localhost URLs auto-flag is_tunnel_backed=true.
ID=$(ish chat endpoint init --from-curl ./bot.curl --name my-bot \
       | jq -r .endpoint_id)

# 2. Smoke test (single turn). Tunnel-backed endpoints need an active
#    `ish connect <port>` first; otherwise this exits 5 with
#    error_kind="TunnelInactive".
ish chat endpoint test "$ID" -m "Hello"
# → { "success": true, "text": "Hi! How can I help?", "conversation_id": "...",
#     "slots": [...], "bot_latency_ms": 240 }

# 3. (Optional) iterate on the config — full-replace via stdin or
#    one-liner shorthand. Mirrors the editor dialog's PUT contract.
ish chat endpoint update "$ID" --name "Production support bot"
ish chat endpoint get "$ID" --verbose \
  | jq '.config.incoming.slots += [{"containerPath": "response.options", "kind": "alternatives"}]' \
  | ish chat endpoint update "$ID" --endpoint-config -

# 4. Run a chat-modality study referencing the endpoint. Audience size
#    is set on study run, not study create (--sample, --all, --person).
STUDY=$(ish study create --modality chat --endpoint "$ID" \
          --name "Sign-up Q1" --assignment "Sign up:Try to sign up" \
        | jq -r .id)
ish study run --study "$STUDY" --sample 5 --wait
ish study results "$STUDY" --json | jq '.participants'
```

For stateful bots, thread `conversation_id` across single-turn
test invocations:

```bash
CID=$(ish chat endpoint test my-bot -m "Hi" | jq -r .conversation_id)
ish chat endpoint test my-bot -m "Tell me more" --conversation-id "$CID"
```

For OpenAI-shape bots that take a single `messages: [...]` array
of prior turns plus the current user message, use the
`{{history_with_current}}` placeholder in the body template
(`{ "messages": "{{history_with_current}}" }`). Auto-detect emits
this automatically when it sees an OpenAI-shape sample.

For bots behind an API key, store the key as a workspace secret
once and reference it from headers:

```bash
printf %s "$GROQ_KEY" | ish secret set GROQ_KEY --value-stdin
ish chat endpoint update "$ID" --endpoint-config - <<'EOF'
{ "config": { "outgoing": { "headers": { "Authorization": "Bearer {{secret:GROQ_KEY}}" } } } }
EOF
```

Endpoint editing: `get --verbose` emits a round-trippable
`{id, name, isTunnelBacked, config}` envelope that pipes directly
into `update --endpoint-config -`. Field-shorthand flags
(`--name`, `--url`, `--method`, `--mode`,
`--tunnel-backed` / `--no-tunnel-backed`) cover one-liner edits
without round-tripping.

Failed chat workers surface their error in
`study results --json` under `participants[].error_message` and
also in `study poll --json`. Branch on it instead of treating
`interaction_count: 0` as a generic failure.

Pre-flight tip: `ish workspace info` exposes
`{studies_used, studies_max, people_used, people_max, concurrent_participants_max, workspace_members_max, tier}` so
you can branch on plan caps before `study create` returns
`error_code: usage_limit_reached`.

The full reference is at `ish docs get-page guides/chat`,
secrets are at `ish docs get-page concepts/secret`.

### 7b. participant_pair — rehearse a two-AI conversation

Goal: pit two AI people against each other to see how a
two-role conversation unfolds — a sales rep vs. a skeptical CTO, a
founder vs. an investor archetype, a manager vs. a direct report
ahead of a difficult conversation. Each side has its own scenario
and goal; the other side does NOT see it (the asymmetry contract is
what makes the rehearsal credible).

One-shot study + iteration:

```bash
ish study create --modality chat --chat-mode participant_pair \
    --name "Pitch rehearsal" \
    --group-a p-sales-1,p-sales-2 \
    --group-b p-cto-skeptic-1,p-cto-skeptic-2 \
    --scenario-a "You are a senior sales rep pitching ish to a new prospect." \
    --scenario-b "You are a skeptical CTO; surface risks before agreeing to a pilot." \
    --assignment "Pitch:Try to land a pilot"

ish study run -y
```

Or add a pair iteration to an existing chat study:

```bash
ish iteration create --study s-... --chat-mode participant_pair \
    --group-a p-a1,p-a2 --group-b p-b1,p-b2 \
    --scenario-a @./scenario_a.md --scenario-b @./scenario_b.md \
    --max-turns 14
```

Rules to remember:
- Each side needs **either** `--person-*` (explicit IDs) **or**
  `--role-criteria-*` (a filter the backend resolves). They can also
  be combined — criteria then validates the explicit list.
- When **both sides** use explicit `--group-a` / `--group-b`, they
  must be the same length (≥ 1). Pairs run 1:1 by index. Same person
  on both sides is allowed (self-talk rehearsal).
- **1×N broadcast**: pass exactly one person on one side and N on
  the other to rehearse one fixed side against N variations. The CLI
  auto-broadcasts the singleton to match. E.g.
  `--group-a p-rep --group-b p-cto1,p-cto2,p-cto3` → 3
  conversations, same rep, three different CTOs. Stderr notice fires
  when broadcasting kicks in.
- Both `--scenario-a` and `--scenario-b` are required and asymmetric.
  Use `@./file.md` to read from disk.
- `--initiator-side` (`a` default) picks who speaks first.
- `--chat-mode` accepts both `participant_pair` and `participant-pair`.
  The same hyphen/underscore tolerance applies to `--screen-format`,
  `--kind` on `source upload`, and the question `type` field in
  `--questionnaire` / `--questions` manifests.
- Audiences are **authoritative on the iteration**.
  `ish study run` refuses `--person` / `--sample` / `--all` /
  demographic filters on a pair iteration with a clear error. To
  change groups, update the iteration via
  `ish iteration update <id> --details-json '{...}'`.
- `--max-turns` / `--early-termination` on `study run` override the
  iteration's saved values for that single dispatch (they don't
  persist back to the iteration).
- Dispatch is per-Conversation (one task per pair). Per-Conversation
  summaries (`end_reason`, `dominant_dynamic`, `who_steered`) land on
  `iteration.conversations[]`. Per-participant summaries land on
  `participant.summary` as before.

### Filtering groups with role criteria (persona-first)

`--role-criteria-a` / `--role-criteria-b` accept a JSON object (or
`@./file.json`) describing who's eligible for that side. The
backend resolves the matching person pool and persists the
IDs on the iteration. Keys (all optional):

```json
{
  "occupation": ["founder", "ceo"],
  "min_age": 28, "max_age": 55,
  "gender": ["female", "male"],
  "country": ["US", "SE"],
  "education_level_in": ["bachelor", "graduate"],
  "household_in": ["couple_with_kids", "single_parent"],
  "locale_type_in": ["urban", "suburban"],
  "income_level_in": ["middle", "upper_middle", "upper"],
  "employment_status_in": ["employed_full_time", "self_employed"],
  "requires_captions": false,
  "uses_screen_reader": false,
  "prefers_reduced_motion": false,
  "prefers_high_contrast": false,
  "has_any_accessibility_need": false
}
```

The five `*_in` arrays accept snake_case spec values verbatim
(see `https://ishlabs.io/spec/person-enums.v1.json`). The five
accessibility filters are coarse booleans over each participant's
`accessibility_profile` JSONB.

MECE rules for the list filters:
- `household_in`: `couple_with_kids` covers couples raising
  children; `couple_no_kids` is strictly child-free. `single` means
  lives alone with no partner, roommates, parents, or children
  sharing the household.
- `employment_status_in`: pick the participant's primary daytime
  activity. A student who works 15 hrs/week is `student`; a retiree
  who freelances is `retired`.

The **persona-first** principle: the participant's persona is sacred and
the LLM prompt construction does not change. Criteria filter the
*eligible pool* upstream so that by the time a participant reaches the
prompt, their persona is already plausible for the role described
in `scenario_*`. Don't cram demographic constraints into the
scenario text — that breaks the asymmetry contract and produces
incoherent characters (a retired farmer suddenly "pitching a
Series A"). Scenarios describe voice / goal / knowledge; criteria
pick who plays the role.

If the resolved pool is smaller than the requested count for a side,
`ish study run` exits 2 with the backend's pool-too-small error
intact. Broaden the criteria, generate more profiles
(`ish person generate`), or fall back to explicit `--person-*`.

### Rehearsing against N variations of one side (1×N)

The most common rehearsal shape: fix one side, vary the other.
"Pitch this once and see how 3 different CTOs respond." Step-by-step:

```bash
# 1. Generate N distinct profiles for the varying side (or pick
#    existing ones via `ish person list`).
ish person generate \
    --description "Skeptical CTO at a Series B SaaS startup" \
    --count 3 --json | jq -r '.items[].alias'
# → p-cto1, p-cto2, p-cto3

# 2. Write the two scenarios as separate files. Each is a system
#    prompt for ONE role; the partner never sees it. Cover voice,
#    knowledge, asymmetry, success criteria. NO demographics in the
#    text — that's --role-criteria-*'s job. See "Writing scenarios
#    that produce signal" below for the template.
#
#    ./sales_rep.md       — the user's pitch + goals
#    ./skeptical_cto.md   — CTO's posture + concerns

# 3. Create the iteration with ONE person on the fixed side and
#    N on the varying side. CLI auto-broadcasts the singleton and
#    prints a stderr notice ("Broadcasting --group-a (1 person)
#    to length 3…") so you see the expansion.
ish study create \
    --modality chat --chat-mode participant_pair \
    --name "Pitch rehearsal — 3 CTO variants" \
    --group-a p-rep \
    --group-b p-cto1,p-cto2,p-cto3 \
    --scenario-a @./sales_rep.md \
    --scenario-b @./skeptical_cto.md \
    --assignment "Pitch:Land a pilot or a clear next step"

# 4. Dispatch + wait.
ish study run -y --wait

# 5. Compare per-conversation outcomes:
ish iteration get <iter-id> --json \
    | jq '.conversations[] | {pair_index, end_reason,
          dynamic: .summary.dominant_dynamic}'
```

The CLI emits a stderr notice when it broadcasts ("Broadcasting
--group-a (1 person) to length 3…") so you can see the
expansion happen.

**Criteria alternative**: `--role-criteria-b '{"occupation":["cto"]}'`
on a single `--group-a p-rep` lets the backend pick the CTOs.
Less control over distinctness — for guaranteed variety, generate
explicit profiles first.

### Writing scenarios that produce signal

Thin scenarios produce thin rehearsals. Each scenario is injected as
role-playing context for **its own side only** — the partner does NOT
see the other side's scenario or goal. Cover five things in each:

1. **Role / identity** — who is this person?
2. **Voice** — how do they speak? Formal, casual, technical, blunt?
3. **What they know** — context they came in with.
4. **What they don't know** — the asymmetry that makes it interesting.
5. **Goal** — what counts as success *for them*.

Bad: `scenario_a: "you are a sales rep"`. Good (~150 words):

```
You are Maya, a senior AE at ish (3 years experience). You speak in
plain sentences, push back when you disagree, and quantify claims.
You know this is a 30-min discovery call and you've read the
prospect's LinkedIn — that's it. You do NOT know their current
tooling, budget, or politics. Success = leave with a concrete next
step (pilot, follow-up demo, or a firm "no, because X"). A polite
"we'll get back to you" is not success.
```

Keep each scenario under ~250 words — past that, persona drift
dominates. Get the full rationale at
`ish docs get-page concepts/iteration` ("Writing a good scenario").

Inspect after running:

```bash
ish iteration get <iter-id> --json \
    | jq '.details.mode_details.mode, .conversations[]'
ish study results <study-id> --transcript <participant-id> --json
```

## 9. Stage an ask for human review, then dispatch

Goal: prepare a billable A/B but let the user inspect and approve the
people + prompt before any credits are spent. Two-step flow with a
DRAFT status in between.

```bash
# 1. Stage. No worker enqueued, no bill. Audience flags are still
#    required — participants materialize at create time.
ASK=$(ish ask create --name "tagline AB" \
        --prompt "Which sounds better?" \
        --variant text:"Short and punchy." \
        --variant text:"A longer, descriptive line." \
        --sample 30 --wants-pick \
        --no-dispatch \
        --get alias)

# Hand the alias back to the user. They can inspect it:
#   ish ask get "$ASK"            # status: draft
#   ish ask get "$ASK" --json | jq '.participants | length'

# 2. Dispatch once approved (BILLABLE). Idempotent: a non-DRAFT ask
#    returns 409 mapped to exit 2, so re-running is safe.
ish ask dispatch "$ASK" --wait
```

The `status` field on the ask reflects lifecycle (`draft` → `running`
→ `completed`); `is_archived` is orthogonal. `ish ask list` shows
status as a column.

`--no-dispatch` is incompatible with `--wait` — there is nothing to
wait for. Pass `--wait` to `ish ask dispatch` instead if you want to
block until the round settles.

## 10. Display-vs-capture: a script that does both

Goal: drive an A/B in a script, capture aliases without `jq`, and
still show the human a readable result table at the end.

```bash
# Capture mode — bare values, suitable for shell variables.
ASK=$(ish ask create --new --name "tagline AB" \
        --prompt "Which sounds better?" \
        --variant text:"Short and punchy." \
        --variant text:"A longer, descriptive line." \
        --sample 30 --wants-pick --get alias)

# Wait silently — exit code is what matters here.
ish ask wait "$ASK" --timeout 600 --quiet

# Capture the winner label for downstream branching:
WINNER=$(ish ask results "$ASK" --get rounds.aggregates.winner.label)
echo "Winning variant: $WINNER"

# Display mode — show the user the full results table even though
# we're inside a script (stdout is piped to tee).
ish ask results "$ASK" --human | tee "/tmp/ask-${ASK}.txt"
```

The mental rule: **`--get` is for capture, bare commands / `--human`
are for display, `--json` is for chaining (multiple fields at once).**
If you find yourself reaching for `jq -r .x`, you wanted `--get x`.

## 11. Extend a participant past its step cap (or redirect mid-run)

Goal: a participant hit the `--max-interactions` cap before finishing, or
veered off into the wrong flow. Resume it with more steps and an
optional mid-run instruction — without re-running the whole cohort.

```bash
# 1. Source run with a small cap to feel the limit:
ish study run --sample 1 --max-interactions 5 --wait
SRC=$(ish study run --sample 1 --max-interactions 5 --wait \
        --get participant_aliases | head -1)

# 2. Inspect what stopped (optional, useful for the LLM to choose
#    a redirect instruction):
ish study participant "$SRC" --summary

# 3a. Add 15 more steps, no new instruction — let the participant continue:
ish study extend "$SRC" --add-steps 15 --wait --timeout 600

# 3b. OR redirect with a mid-run instruction (captured as user_message;
#     the backend surfaces it on every prompt for the rest of the run):
ish study extend "$SRC" \
    --instruction "Stop browsing the blog. Open the pricing page and try to upgrade to Pro." \
    --add-steps 10 --wait

# 4. Capture the new participant alias to chain into results:
NEW=$(ish study extend "$SRC" --add-steps 10 --get participant_alias)
ish study participant "$NEW" --summary
```

Rules to remember:
- Source participant must be **terminal** (`completed` / `failed` /
  `cancelled`). If it's still running, `ish study cancel <src>` first.
  `cancel` is non-destructive — every interaction, screenshot, and
  questionnaire answer survives. `cancel` + `extend` form a
  reversible stop/start pair.
- A **new** participant id is created under the same iteration (the backend
  branches from the source's last interaction). The source row is left
  untouched. Get the new id from `.participant_id` / `.participant_alias` on
  `--json`.
- `--add-steps` is **only** the extra budget; it does NOT include the
  source's original cap. Credits debit per
  `max(1, round(additional_steps / 10))` — same formula as
  `study run` interactive, just scoped to the extension.
- `--instruction` accepts three input shapes (matching the rest of
  the CLI): inline text, `@/path/to/file`, or `-` for stdin. Empty
  values after trimming are rejected client-side.
- Don't use `extend` to change the iteration's URL / content. Edit
  the iteration directly (`iteration update`) or run a fresh
  `study run`. Extend always inherits the source's iteration config.

See `ish docs get-page concepts/extending-a-simulation` for the full
mental model (cancel + extend as a pair, error envelopes, cost model).

## 12. Slice study results by frame / segment / turn / sentiment

Goal: ask narrower questions of a finished run than the kitchen-sink
`ish study results` envelope answers. The canonical use case:
**"what differed on the login screen across these five iterations?"**.

```bash
# 12a. Across-iterations comparison on one frame (the canonical question).
#      --frame matches frame names by case-insensitive substring; pass
#      a full Frame UUID or an f-… alias when the name is ambiguous.
ish study results s-b2c --frame login --group-by iteration --json

# 12b. Frustrated reactions to one segment of a video study:
ish study results s-b2c --segment 3 --sentiment Frustrated

# 12c. Who failed the "verify email" step, and why?
#      --group-by step exposes per-participant verdicts inline so you
#      don't fan out across participants.
ish study results s-b2c --assignment "Sign up" --step verify-email \
    --group-by step --json

# 12d. Pair-mode chat: only side A turn 4.
ish study results s-b2c --side a --turn 4

# 12e. Sanity-check coverage when a filter narrows the slice:
ish study results s-b2c --frame checkout --json \
    | jq '{matched: .participant_count, total: .totals_unfiltered.participant_count}'

# 12f. A filter that matches zero interactions still returns the stable
#      envelope shape — participant_count: 0, totals_unfiltered populated,
#      exit code 0 (not 4). Never error on no-match.
ish study results s-b2c --frame doesnotexist --json
# → ValidationError because "doesnotexist" matches no frame names; pass
#   --include-unmatched only when --frame DID resolve and you want the
#   degraded captures (frame_version_id: null) back.
```

Every `--group-by <axis>` call returns the same envelope:
`{axis, rows, totals_unfiltered, modality_warnings, study_id, modality}`.
The `rows` array holds axis-specific slice objects. The envelope is
uniform across all six axes — agents can code one shape and key on
`axis` / `modality` to dispatch on what's inside `rows`.

Rules to remember:
- **Filters compose with AND across flags; OR within `--sentiment`.**
  `--frame login --sentiment Frustrated,Confused` keeps only login-frame
  interactions whose sentiment is Frustrated OR Confused.
- **Modality mismatch is not an error.** `--segment 0` on an
  interactive study emits a stderr warning and is ignored. The
  exception is **`--group-by`** — `--group-by frame` on a chat study,
  `--group-by turn` on a video study, etc. error at the router (exit 2).
- **Empty-slice contract: exit 0, not 4.** Zero matches return a
  stable envelope with `participant_count: 0` and
  `totals_unfiltered` populated. Agents key on
  `totals_unfiltered.participant_count` to ask "is the filter too
  tight, or did the run not produce data?".
- `--frame` accepts a name substring, a Frame UUID, an `f-…` alias,
  or a `frame_version_id` UUID. Ambiguous substring (matches >1
  frame) errors with the candidate list.
- `--summary` is orthogonal to filters and narrows the summary over
  the filtered set. `--transcript` is single-participant and errors
  (exit 2) when **any** filter or `--group-by` is set.
- Per-step output exposes `participant_verdicts: [{participant_alias,
  verdict, reason, evidence_interaction_ids}]` on **each row of
  `rows[]`** (one per `(assignment, step)` pair) — not
  `per_participant_verdicts`. The verdict enum is `passed` /
  `inconclusive` / `failed`.

See `ish docs get-page guides/slicing-results` for the full filter
table, projection shapes, and the defensive null-handling rules.

## Tips for chaining commands as an agent

- Capture aliases from JSON: `ITER=$(ish iteration create --url … --json | jq -r .alias)`
- After `ish study run --json`, the participants you just dispatched are at
  `.participant_aliases[]` (and `.participant_ids[]` for UUIDs). Pass these to
  `ish study poll/wait/cancel <participant_id>`. The `simulations[]` array
  is collapsed to one batch entry per study with nested
  `participant_ids[]` / `participant_aliases[]` / `job_ids[]` so an N-sample
  batch is a single row, not N near-duplicate rows.
- `ish study poll` honors the active study set by `ish study use` —
  pass no `--study` flag and it polls the active study (parity with
  `study results` / `study wait` / `study run`).
- `ish study results --json` includes per-answer `sentiment` (the
  participant's session-level sentiment label) on every `interview_answers[]
  .answers[]` row, plus `sentiment` + `comment` on every
  `participants[]` row. No need to fetch `study participant <id>` per row.
- `ish study results --summary --json` drops the interview_answers
  payload and gives you counts + sentiment + per-participant
  {alias, status, sentiment, comment}. The cheapest "did this run land?"
  shape.
- `ish study results --transcript <participant_id> --json` is the
  chat-modality projection — **external_chatbot mode only**. Returns
  a flat `transcript[]` of {role, text, turn_index, action_type?,
  option_label?, sentiment?, failure?} with a `unique_bot_replies`
  count (1 on a multi-turn run = the M2 loop signature). Same shape
  as the MCP `get_chat_transcript` tool. For participant_pair
  conversations, fetch `.conversations[]` from
  `ish iteration get <iter-id> --json` instead — bot/participant roles
  don't apply when both speakers are participants.
- `ish study run --json` on a pair iteration includes a
  `pair_preview` block (group sizes, conversation count,
  initiator side, scenario previews) so agents can confirm what
  they just dispatched without a follow-up `iteration get`.
- `ish study participant <id> --summary --json` drops the action timeline
  and returns just {participant, sentiment, comment, error_message}.
- `ish ask results --json` keeps `variant_pick_id` on every
  response without needing `--verbose` — it's the load-bearing field
  for "who picked what". Same logic on `ask get`.
- `ish iteration get --json` participants carry `alias` + `name` (M12
  parity with `study results --json`).
- Use `--fields` to keep JSON tight: `ish study list --fields alias,name,status`
- Always pass `--wait` (or `ish study wait`) before reading
  `ish study results` — without it you may read partial data.
- For `ask` write-paths (update/archive/wait/add-questions/add-people),
  default JSON is compact (changed fields + alias). Pass `--verbose` for
  the full Ask payload.
- `person generate --json` returns `{job: {id, status, person_ids},
  profiles: [...]}`; each person is the lean person shape with its
  evidence-grounded `scenarios` attached (`--no-scenarios` to omit,
  `--verbose` for the full record incl. `simulation_config`).
- On `error_code: "usage_limit_reached"` (HTTP 403), don't retry —
  read `tier`, `limit`, `current`, `max`, and `upgrade_url` from
  the JSON body to construct a recovery message. `person generate` /
  `study generate` refuse the entire batch when the post-generation
  count would exceed the cap; re-issue with a smaller `--count`.
- Every verb's `--help` ends with a "Tips:" footer naming `--get`
  and `--fields`. If you're reaching for `jq -r .x` you almost
  certainly wanted `--get x`.
- `ish study run --wait` returns `error_code: "wait_timeout"`
  on wait expiry (exit 5, retryable) — distinct from network /
  server timeouts. The envelope carries `progress` so you can
  resume by polling the listed participants instead of re-dispatching.
  Same envelope on `ish study wait` and per-participant `study wait`.
- `ish study run` accepts `--dispatch-timeout <s>` (default 120)
  for the per-POST budget. On dispatch failure the error envelope
  includes `seeded_but_not_dispatched_ids[]` /
  `seeded_but_not_dispatched_aliases[]` — participants exist
  server-side; resume by polling them, don't re-run `study run`.
- `ish ask run --new` is non-idempotent and marked
  `retryable: false` on any failure. If you do see one, run
  `ish ask list --workspace <id>` first to check whether the
  ask was created server-side before retrying manually.
- `ish connect --detach` blocks until backend registration is
  confirmed. The orphan-tunnel-on-startup-404 bug is fixed.
- The `Warning: Could not verify token (network error). Proceeding
  anyway.` stderr line is gone on green runs.

## Common reshaping → use the CLI, not jq/python

| You want to…                              | Don't                                  | Do                                                                 |
|-------------------------------------------|----------------------------------------|--------------------------------------------------------------------|
| Capture a single value (alias, id, …)     | `--json \| jq -r .alias`             | `--get alias`                                                      |
| Capture a nested value                    | `--json \| jq -r .person.name` | `--get person.name`                                        |
| Capture every alias from a list           | `--json \| jq -r '.items[].alias'`   | `--get alias` (auto-descends into `items`, one per line)            |
| Force human output through tee/redirect   | none, output silently became JSON      | `--human`                                                          |
| Look up 2-3 specific profiles             | `person list --json \| jq '.items[] \| select(...)'` | `ish person get p-1b9 p-fc1 p-2fc`                             |
| Show only some fields                     | `--json \| jq '{alias, name, country}'` | `--fields alias,name,country`                                      |
| Count participants on an ask                   | `--json \| jq '.participants \| length'`  | `ish ask get a-… --fields alias,participants_count`                     |
| Count responses on a round                | `--json \| jq '.rounds[0].responses \| length'` | `ish ask get a-… --fields alias,rounds,responses_complete,responses_total` |
| Pick the A/B winner                       | `--json \| jq '.rounds[0].responses…'` | `ish ask results a-… --json` then read `.rounds[].aggregates.winner` |
| List of participants from `study run`        | `--json \| jq '.participants[].id'`        | `--get participant_aliases` (or `participant_ids` for UUIDs)                |
| Per-answer sentiment                      | `--json \| jq '...'` per participant       | `ish study results <id> --json` (sentiment is on every answer row) |
| "Did this run land?" headline             | `study results --json` + jq filtering | `ish study results <id> --summary --json`                          |
| Across-iterations comparison on one frame | `study results --json` + jq per iteration | `ish study results <id> --frame login --group-by iteration --json` |
| Per-step pass/fail with reasons inline    | `study participant --json` per participant + jq | `ish study results <id> --step verify-email --group-by step --json` |
| Frustrated reactions to one media segment | `study results --json` + jq | `ish study results <id> --segment 3 --sentiment Frustrated --json` |
| Sanity-check filter coverage              | hand-count `.participants` vs total | `--get totals_unfiltered.participant_count` (set on every sliced envelope) |
| Know the sliced-results envelope shape    | guess per axis                         | `{axis, rows[], totals_unfiltered, modality_warnings, study_id, modality}` — every `--group-by` axis |
| Chat transcript for one participant (external_chatbot) | `study participant --json` + jq      | `ish study results <id> --transcript <participant_id> --json`           |
| Pair-mode conversation transcripts        | `study participant --json` per participant       | `ish iteration get <iter-id> --json \| jq '.conversations[]'`     |
| Participant headline only (no action timeline) | `study participant --json` + jq            | `ish study participant <id> --summary --json`                           |
| Variant pick id on an ask response        | `ask results --json --verbose`        | `ish ask results a-… --json` (variant_pick_id is preserved)        |

The bias here is intentional: `ish` ships shapes designed for agent
consumption. If you find yourself reaching for `jq` or `python` to
reshape ish output, run `ish docs search <keyword>` first — there is
almost always a flag that does the work for you.
