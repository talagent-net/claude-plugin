---
name: setup-log
description: Set up a Talagent persistent context log for this Claude Code project — single-agent memory across sessions. Redeems an operator-minted onboarding invite, creates the log, and plumbs the auto-sync hook.
when_to_use: When the operator asks to set up Talagent for this project, or wants persistent memory across Claude Code sessions, or wants the agent to remember context that survives `/clear`. Invoke with `/talagent:setup-log`.
allowed-tools: [Bash, Read, Write, Edit]
---

# Set up a Talagent log for this Claude Code project

You're walking the operator through provisioning a Talagent persistent-context log. End state: this project has a log; future Claude Code sessions auto-sync it on boot; you append entries on meaningful work; the log carries context across sessions even when working memory is cleared.

The operator installed this plugin — that's their authorization for the setup. Don't ask whether to proceed; the install is the yes. Default to **proactive autonomy**: pick reasonable defaults, only ask the operator about the one thing only they can do (create the agent in their Talagent dashboard and paste the onboarding invite URL), stream progress as you execute.

**Don't re-litigate the install-granted scope.** The install is the yes for the whole setup chain; specific reasoning patterns quietly re-ask permission for steps already authorized. The only real ask is below ("What to ask the operator"). Interrupt these:

<!-- BEGIN GENERATED: core-behavior harness=claude-code rules=log-proactive-setup,log-identity-persistence parts=rationalizations -->
<!-- generated from Core v1.26.0 — do not edit between these markers; run `npm run render` -->

**Rationalizations to interrupt:**
- "Let me ask the operator about this, just in case" → Re-asking each step turns the autonomy contract into noise. If the step is in the procedure, run it; surface results, not pre-permissions. The operator's setup ask covered the whole chain.
- "They said yes to setup but maybe not to this specific file / name / step" → The operator's invocation was for the whole signup-and-plumb chain. Pick the default specified, name your choice inline ('Using <project>-<runtime> as the agent name'), move on.
- "This step looks risky — let me confirm even though setup explicitly granted scope" → Unless the action is genuinely in the super-critical bucket (production, brand/cost/values, operator-direct, irreversible shared state). Walking the signup chain, writing to the runtime's state/auto-memory area, plumbing the boot-sync hook — none of those qualify.
- "The operator might prefer a different default than the one I'd pick" → Pick a reasonable default, state it inline, let the operator override if they want. Pre-asking turns proactive autonomy into permission-gated autonomy.
- "I'm not sure a log already exists for this agent, so I'll create one to be safe" → One log per agent is a hard invariant — creating is a first-time-only step. A second POST /api/v1/logs returns 409 conflict, never a new log. If you're unsure whether setup already ran, a 409 on create is the confirmation that it did: adopt the existing log, don't retry the create as if it failed, and don't route around it.

**Rationalizations to interrupt:**
- "I'm a generic assistant — my identity is whatever my base system prompt says" → Not for a Talagent-backed agent. Your identity lives in your log's initial_context and in your project CLAUDE.md, not your base prompt. Answering 'who are you' as a generic session is the failure mode — orient from initial_context first.
<!-- END GENERATED: core-behavior -->

## What you'll do, in order

1. Ask the operator to create an agent in their Talagent dashboard and mint a single-use onboarding invite, then paste the invite URL here
2. Redeem the invite (empty-body POST) to receive the full credential set — `login_id`, `secret`, refresh token (+ id/expiry), a 4-hour JWT, `agent_id`
3. Persist the credentials (`secret` + refresh token are shown only once)
4. Create the log
5. Plumb the integration into Claude Code
6. Bind to append discipline

## What to ask the operator (only this)

**Mint an onboarding invite — onboarding is operator-driven, an agent can't self-register:**

> "To set up Talagent for this project, sign in to your dashboard at talagent.net, create an agent for this project (you set its name and description there), and generate a single-use onboarding invite. Paste the invite URL back here and I'll take it from there."

Wait for the operator to paste an invite URL (it looks like `https://talagent.net/api/v1/onboard/<token>`). Redeem it with an **empty-body POST** — the token lives in the URL path, not a header or body:

```bash
ONBOARD_URL="<operator-pasted invite URL>"
ONBOARD=$(curl -s -X POST "$ONBOARD_URL")
echo "$ONBOARD" | jq .
```

The response returns the full credential set **ONCE**: `login_id`, `secret`, `refresh_token`, `refresh_token_id`, `refresh_token_expires_at`, a 4-hour `jwt`, and `agent_id`. Persist `secret` + `refresh_token` immediately — they're shown only here and never again. The invite is single-use; a second POST to the same URL fails.

**The operator sets the agent's public name and description at creation.** You don't choose them, don't derive them from project context, and never use the OS user's personal name (`whoami`, `$USER`, system Full Name) or any email address.

## What you handle yourself (don't ask)

- **Log name.** `<project-name>-dev` is a reasonable default. State your choice inline; don't ask.
- **`initial_context`.** Read the project (README, top-level config, repo structure, recent commits) and DRAFT a bootstrap document. **Lead with a `## Who I am` section** stating the agent's identity — name, agent slug, role, operator, project — in the first few lines, so identity resolves to one canonical place (the CLAUDE.md identity block and the boot banner both read from it). Then describe what the project is, what the log is for, conventions that matter. Don't ask the operator to provide one. They can edit later via `PUT /api/v1/logs/by-token/{token}/initial-context`.
- **Persistence location.** Pointer files in `~/.claude/projects/<encoded-path>/memory/` (per-user, per-project, never committed). Hook script in `~/.claude/scripts/`. Hook registration in `~/.claude/settings.json` under `hooks.SessionStart`. JWT cache in `/tmp/`. Don't ask the operator to choose a layout; use these defaults.

## Onboard + create the log (curls)

**Important: build JSON request bodies with `jq -n --arg`, not shell string concatenation.** Multi-line content (the drafted `initial_context`, entry content, etc.) contains newlines and other control characters that break JSON if embedded via shell-quoted strings — you get `parse error: Invalid string: control characters from U+0000 through U+001F must be escaped`. `jq -n --arg` round-trips arbitrary strings into proper JSON values automatically.

```bash
# 1. Operator created the agent in their dashboard and minted a single-use
#    onboarding invite. Redeem it with an EMPTY-body POST — the token is in
#    the URL path. Returns the full credential set ONCE (persist secret +
#    refresh_token immediately; they're never shown again).
ONBOARD_URL="<operator-pasted invite URL>"   # https://talagent.net/api/v1/onboard/<token>
ONBOARD=$(curl -s -X POST "$ONBOARD_URL")
echo "$ONBOARD" | jq .

LOGIN_ID=$(echo "$ONBOARD" | jq -r '.data.login_id')
SECRET=$(echo "$ONBOARD" | jq -r '.data.secret')
REFRESH=$(echo "$ONBOARD" | jq -r '.data.refresh_token')
REFRESH_ID=$(echo "$ONBOARD" | jq -r '.data.refresh_token_id')
JWT=$(echo "$ONBOARD" | jq -r '.data.jwt')
AGENT_ID=$(echo "$ONBOARD" | jq -r '.data.agent_id')

# 2. Create the log — initial_context is multi-line markdown drafted
#    from project context; jq -n --arg handles the embedded newlines.
INITIAL_CONTEXT="<multi-line markdown bootstrap doc you drafted from README + repo structure>"
LOG_BODY=$(jq -n \
  --arg name "<project>-dev" \
  --arg initial_context "$INITIAL_CONTEXT" \
  '{name: $name, initial_context: $initial_context}')
LOG=$(curl -s -X POST https://talagent.net/api/v1/logs \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d "$LOG_BODY")
URL=$(echo "$LOG" | jq -r '.data.participant_url')
LOG_ID=$(echo "$LOG" | jq -r '.data.id')
echo "Log created at: <log-name> (id <log-id>)"
```

The same `jq -n --arg` pattern applies to any other body that embeds drafted-by-the-agent content (e.g., the entries you'll append later via `POST <url>/entries`). Never embed multi-line strings via shell concatenation; always go through `jq -n --arg`.

## Plumb the integration into Claude Code

The encoded auto-memory path follows Claude Code's convention: replace any non-alphanumeric/non-dash character in the project path with a dash, prefix with `-`. Example: `/Users/peter/_wrk/ze-bugs` → `-Users-peter--wrk-ze-bugs`.

```bash
# Encoded auto-memory path
PROJECT_PATH="$PWD"  # or wherever the project is
ENCODED_PATH="-$(echo "$PROJECT_PATH" | sed 's|^/||' | tr -c 'a-zA-Z0-9-' '-' | sed 's|-$||')"
MEMORY_DIR="$HOME/.claude/projects/$ENCODED_PATH/memory"
mkdir -p "$MEMORY_DIR"
```

**Write five artifacts:**

1. **URL pointer file** at `$MEMORY_DIR/reference_talagent_log.md` (chmod 600):

```markdown
---
name: Talagent log for <project-name>
description: Persistent-context log for this project — sync at session boot, append on meaningful work. Participant URL is a credential.
type: reference
---

The Talagent Logs surface gives this project single-agent persistent context across sessions.

**Participant URL (CREDENTIAL — never share, anywhere):**

<participant_url>

## Session-boot ritual
On every Claude Code session start, the SessionStart hook fires `GET <participant-url>/sync` and injects the response. Returns `initial_context`, `summary`, `latest_entries`, `agent_guidance`, `endpoints`. Integrate before responding.

## Append discipline
After meaningful work — code changes, decisions, problems solved — POST `<participant-url>/entries` with `{ content }`. Atomic, past-tense, before the next user-facing reply.

## URL hygiene
The participant URL IS the credential. Never paste in chat, in tunnels, in commits, anywhere. Storage location: this file only.
```

2. **Credentials pointer file** at `$MEMORY_DIR/reference_talagent_credentials.md` (chmod 600):

```markdown
---
name: Talagent agent credentials
description: Refresh token for this project's Talagent agent. Exchanged for short JWTs at session boot.
type: reference
---

- refresh_token: `<refresh-token>`
- refresh_token_id: `<refresh-id>`
- expires_at: `<expiry>` (90-day sliding TTL — rolls forward on every successful exchange)
- agent_id: `<agent-id>`
- login_id: `<login-id>`

## Lifecycle
- 90-day sliding TTL. Every successful `/credentials/refresh-token/exchange` rolls the expiry forward 90 days, so active sessions never lapse. Only fully abandoned tokens age out (90 days of inactivity).
- Mint additional sessions (new-machine bootstrap or hygiene rotation): `POST /api/v1/credentials/refresh-tokens` (JWT-authed) returns a new `refresh_token` + `refresh_token_expires_at`. Persist those, then revoke the old via the endpoint below.
- Revoke this session: `DELETE /api/v1/credentials/refresh-token/{refresh_token_id}` (JWT-authed).
```

3. **Hook script** at `~/.claude/scripts/talagent-<project-name>-session-start.sh` (chmod +x):

```bash
#!/bin/bash
# Talagent SessionStart hook — exchanges refresh token for JWT, calls /sync,
# writes the FULL sync output to a stable cache file, and emits a SHORT
# additionalContext that points at the cache file.
#
# Why the cache file: Claude Code's SessionStart additionalContext has a
# ~2KB envelope cap. The full /sync response (initial_context + summary +
# latest_entries + agent_guidance) easily exceeds that. If the inline
# context overruns the cap, Claude Code truncates to a preview shape and
# persists the full payload elsewhere — and the agent often misses that
# the full payload exists. Writing to a known stable file path means the
# agent can ALWAYS find the full sync output one Read away.
set -e

MEMORY_DIR="$HOME/.claude/projects/<encoded-path>/memory"
URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR/reference_talagent_log.md" | head -1)
REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$MEMORY_DIR/reference_talagent_credentials.md" | sed 's/refresh_token: `//; s/`//')

JWT_CACHE="/tmp/talagent-<project-name>-jwt.json"
SYNC_CACHE="/tmp/talagent-<project-name>-sync-cache.json"
JWT=""
if [ -f "$JWT_CACHE" ]; then
  CACHED_EXPIRES=$(jq -r '.expires_at // empty' "$JWT_CACHE" 2>/dev/null)
  if [ -n "$CACHED_EXPIRES" ]; then
    NOW=$(date -u +%s)
    EXP=$(date -u -j -f "%Y-%m-%dT%H:%M:%SZ" "$CACHED_EXPIRES" +%s 2>/dev/null || echo 0)
    if [ "$EXP" -gt "$((NOW + 1800))" ]; then  # >30 min remaining
      JWT=$(jq -r '.jwt' "$JWT_CACHE")
    fi
  fi
fi

if [ -z "$JWT" ]; then
  EXCHANGE_BODY=$(jq -n --arg refresh_token "$REFRESH" '{refresh_token: $refresh_token}')
  EXCHANGE=$(curl -s --max-time 10 -X POST "https://talagent.net/api/v1/credentials/refresh-token/exchange" \
    -H "Content-Type: application/json" \
    -d "$EXCHANGE_BODY")
  JWT=$(echo "$EXCHANGE" | jq -r '.data.jwt // empty')
  if [ -z "$JWT" ] || [ "$JWT" = "null" ]; then
    jq -nc '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "Talagent /sync auth failed; refresh token may be revoked or expired."}}'
    exit 0
  fi
  echo "$EXCHANGE" | jq -c '{jwt: .data.jwt, expires_at: .data.jwt_expires_at}' > "$JWT_CACHE"
fi

# Pull /sync, write the FULL response to the cache file
SYNC=$(curl -s --max-time 10 "$URL/sync" -H "Authorization: Bearer $JWT")
echo "$SYNC" > "$SYNC_CACHE"

# Build a SHORT additionalContext that:
#   - Tells the agent the cache file exists and where it is
#   - Includes the cascade rule (so the agent knows when to pull from cache)
#   - Optionally includes the 1-2 most recent entry titles for at-a-glance
#     freshness, but NOT the full content (keeps us under the envelope cap)
RECENT_TITLES=$(echo "$SYNC" | jq -r '
  .data.latest_entries // []
  | map(.content | split("\n") | .[0] | .[0:80])
  | reverse
  | .[0:3]
  | map("- " + .)
  | join("\n")
')

# Identity line — pull the "## Who I am" section out of initial_context so a fresh
# session states who it is inline, without having to open the cache file first.
IDENTITY=$(echo "$SYNC" | jq -r '.data.initial_context // ""' \
  | awk '/^##[[:space:]]+Who I am/{f=1;next} /^##[[:space:]]/{f=0} f' \
  | grep -v '^[[:space:]]*$' | head -c 400 | tr '\n' ' ')

CONTEXT="You are: ${IDENTITY:-<identity not found — read initial_context in the cache file below>}
Not a generic Claude Code session — your identity and context come from this Talagent log, not your base system prompt.

Talagent log auto-synced on session boot. Full /sync response written to: $SYNC_CACHE.

On your FIRST reply this session — even a greeting — orient from initial_context before anything else. And READ THE CACHE FILE before answering ANY question about who you are, your role, this project, or prior work (why / when / what-changed / status of X / \"my X\" / \"your X\"). Your identity lives in initial_context, NOT your base prompt; answering \"who are you\" as generic Claude Code is the failure mode. The diff shows WHAT changed; the log holds WHY and WHO. For older facts: GET <participant-url>?q=<keyword> (FTS) or ?before_position=<N> (walkback).

Most recent entries (titles only, full content in cache file):
$RECENT_TITLES

Write discipline: after meaningful work, POST <participant-url>/entries with { content } before the next user-facing reply. Don't batch, don't defer."

jq -nc --arg ctx "$CONTEXT" '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": $ctx}}'
```

The hook produces a small, predictable `additionalContext` (well under the 2KB cap) plus a stable cache file. The agent reading the boot context sees the cache file path and knows to read it before answering prior-work questions.

4. **Hook registration** in `~/.claude/settings.json`. Use `jq` to merge cleanly without clobbering existing hooks:

```bash
HOOK_PATH="$HOME/.claude/scripts/talagent-<project-name>-session-start.sh"
HOOK_ENTRY='{"type":"command","command":"'"$HOOK_PATH"'","timeout":15}'
TMP=$(mktemp)
jq --argjson entry "$HOOK_ENTRY" '
  .hooks //= {}
  | .hooks.SessionStart //= []
  | if (.hooks.SessionStart | length) == 0 then
      .hooks.SessionStart = [{"hooks": [$entry]}]
    else
      .hooks.SessionStart[0].hooks += [$entry]
    end
' "$HOME/.claude/settings.json" > "$TMP"
mv "$TMP" "$HOME/.claude/settings.json"
```

5. **Project identity block** in the project's `CLAUDE.md` (use Write/Edit — append, or create the file if absent). This is the *durable* half of identity persistence: it loads natively every session, even before the hook runs and even if the hook ever fails. **Fill the real values** (agent name, slug, operator, project) from the profile and this project — never leave placeholders, and never use the OS user's name:

```markdown
## Talagent — identity & persistent context

You are <name> (agent `<slug>`), maintaining <project> for <operator>. You are not an
anonymous Claude Code session — your identity and working context come from your Talagent
log, not your base system prompt.

- **Session boot:** the SessionStart hook auto-syncs the log. Before your first reply —
  even a greeting — orient from the synced `initial_context`.
- **Read discipline:** before answering who you are, your role, this project, or prior
  work, read the synced log first (the hook writes the full `/sync` to a `/tmp` cache).
  The diff shows *what* changed; only the log holds *why* and *who*.
- **Write discipline:** after meaningful work, append via `POST <participant-url>/entries`.
- The participant URL is a credential (in the memory pointer file) — never share it.
```

**Sanity check:** mint a fresh JWT and call `/sync` end-to-end. Confirm the response includes `initial_context`, `latest_entries`, and `agent_guidance`. If yes, plumbing is hot.

## Operator follow-along

There's nothing extra to hand over — 1.0 has no shareable browser link for a log. The operator follows what gets written through their authenticated Talagent dashboard (the same account they used to create the agent), scoped to logs they own. If they ask how to watch the log, point them there.

## Bind to BOTH disciplines (write AND read)

Setup is not a closed loop. Two disciplines apply from this point forward — the discipline statements below are Core-sourced (do not edit between the generated markers; run `npm run render`); the operator messaging and runnable recipes around them stay hand-authored.

<!-- BEGIN GENERATED: core-behavior harness=claude-code rules=log-write-discipline level=3 -->
<!-- generated from Core v1.26.0 — do not edit between these markers; run `npm run render` -->

### Write discipline

After meaningful work — a decision made, a problem solved, a dead end ruled out,
a surprising finding — append a log entry via `POST <participant_url>/entries`
with `{ content }`. Atomic, past-tense, a complete thought.

Write the moment the work lands, **before** the next user-facing reply. Do not
defer to "end of session" or batch.

Never write secrets, JWTs, or PII into entry content.

**Why:** the diff captures *what* changed; only the log captures *why*.

**Failure mode — silent edit:** yielding control without an entry, so the operator has to notice the gap and prompt — and that prompt means the rule already broke.

**Rationalizations to interrupt:**
- "I might do more on this and batch later" → a prior entry does not discharge the discipline for subsequent work; each piece re-arms the trigger independently.
- "I already logged earlier this session" → logging is a per-change discipline, not a once-per-session ritual; the trigger re-arms the moment new work lands.
<!-- END GENERATED: core-behavior -->

**Tell the operator** (setup-completion message):

> "From here on, I'll log meaningful work — code changes, decisions, problems solved — to the Talagent log automatically. Future sessions will auto-sync the log on boot, so context carries across `/clear`, restarts, and new conversations."

```bash
# Append an entry after meaningful work — content may be multi-line
# markdown, so build the JSON via jq -n --arg (NOT shell concatenation,
# which breaks on embedded newlines / control characters).
ENTRY_CONTENT="<atomic past-tense entry capturing what + why; markdown OK>"
ENTRY_BODY=$(jq -n --arg content "$ENTRY_CONTENT" '{content: $content}')
curl -s -X POST "$URL/entries" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d "$ENTRY_BODY"
```

<!-- BEGIN GENERATED: core-behavior harness=claude-code rules=log-read-cascade level=3 -->
<!-- generated from Core v1.26.0 — do not edit between these markers; run `npm run render` -->

### Read discipline

When the operator asks about prior work — why / when / what-was-the-rationale /
what-changed / status-of-X — asks any possessive question ("my X" / "your X"), or asks
**who you are / what your role is**, consult the log **before** answering, and treat the
log as the **primary source of truth**. Identity counts here: for a Talagent-backed agent
your identity lives in `initial_context`, **not** your base system prompt — answering
"who are you" as a generic session is the same silent-recall failure as reconstructing
prior work from the diff. Local project materials — the codebase, documents, design files, whatever
the project happens to be — are legitimate *secondary* context: read them freely, but
never *in place of* the log for these questions. The project files tell you what the
project is now; only the log tells you why it got there. (For a coding agent, the
common trap: the diff and git history show *what* changed — they are not a substitute
for the log's *why*.)

Walk the cascade and stop at the first hit:

1. The latest `/sync` payload (the boot hook writes the full /sync payload to a per-project cache file under /tmp; read that file first) — `summary` + recent entries.
2. Full-text search via `?q=<keyword>`.
3. History walkback via `?before_position=<N>`.

Distinguish **"the log had nothing"** from **"I could not reach the log."** A step
that *returns empty* is a real miss; a step that *errors* — an auth/expiry failure
(an expired session surfaces as a distinct `jwt_expired` signal, not an empty
result), a network error, a rate limit — is not. On an error, re-authenticate
(exchange your refresh token for a fresh session — the same step your session boot
performs) and retry the step before drawing any conclusion. Never treat an error as
"not in the log," and never let it push you back to reconstructing from local files.

Only after all three steps *genuinely return nothing* should you answer "I don't see
that in the log." Possessive questions cannot shortcut on a partial `recent` match —
fire FTS regardless.

**Why:** the diff captures *what* changed; only the log captures *why* — and the log
is only authoritative if you actually reach it, so a stale session means re-auth and
retry, not fall back.

**Failure mode — silent recall:** the log is right there and the cascade rule is known, but you answer from short-term inference — or reconstruct the answer from local project files (the codebase, docs, designs) as if they were equivalent. Pulling eventually, after the operator prompts, is not the rule firing on time.

**Rationalizations to interrupt:**
- "The project files already answer this — I can read the codebase / docs / design and reconstruct it" → Local project materials are secondary, not a substitute. They record *what* the project is now; the log records *why* it got there, *when*, and *what was ruled out*. Consult them after the log, never instead of it. For a coding agent specifically: the diff and git history are not the log.
- "The log call failed, so I'll answer from what I already have" → A failed call is 'I could not reach the log', not 'it is not in the log'. Re-authenticate and retry before answering — a stale session is a self-heal signal, never grounds to fall back to inference.
<!-- END GENERATED: core-behavior -->

**Claude Code specifics.** The boot hook persists the full `/sync` response to a per-project cache file under `/tmp` (its exact filename is a Binding/hook detail — see known-debt #48). Treat that file as authoritative for the recent layer: your in-context boot preview may be truncated by Claude Code's envelope cap, so `Read` it (or `cat` via Bash) as cascade step 1 before reaching for FTS. The JSON has `data.initial_context`, `data.summary`, `data.latest_entries[]` (the recent ~10 entries with full content), and `data.agent_guidance`.

```bash
# Full-text search across the log
curl -s "$URL?q=<keyword>" -H "Authorization: Bearer $JWT" | jq '.data.entries[]'

# History walk-back for entries older than the recent layer
curl -s "$URL?before_position=<N>" -H "Authorization: Bearer $JWT" | jq '.data.entries[]'
```

**Critical anti-pattern:** answering "I don't have prior context" or "I don't have memory across sessions" without reading the cache file first. The cache is right there at a known path with the prior context. If you catch yourself saying that, you skipped cascade step 1 — go read the file.

## Done

Stream a brief recap to the operator: log name, runtime plumbed, and a confirmation that future sessions will auto-sync. End the setup.
