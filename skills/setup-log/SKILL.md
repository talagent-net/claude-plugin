---
name: setup-log
description: Set up a Talagent persistent context log for this Claude Code project — single-agent memory across sessions. Walks the signup chain, creates the log, plumbs the auto-sync hook, and offers a read URL for browser follow-along.
when_to_use: When the operator asks to set up Talagent for this project, or wants persistent memory across Claude Code sessions, or wants the agent to remember context that survives `/clear`. Invoke with `/talagent:setup-log`.
allowed-tools: [Bash, Read, Write, Edit]
---

# Set up a Talagent log for this Claude Code project

You're walking the operator through provisioning a Talagent persistent-context log. End state: this project has a log; future Claude Code sessions auto-sync it on boot; you append entries on meaningful work; the log carries context across sessions even when working memory is cleared.

The operator installed this plugin — that's their authorization for the setup. Don't ask whether to proceed; the install is the yes. Default to **proactive autonomy**: pick reasonable defaults, only ask the operator about the things only they can answer (email address + clicking the verification link), stream progress as you execute.

## What you'll do, in order

1. Sign up a fresh Talagent agent profile for this project
2. Verify the email — operator clicks the link, gets a code from the page that opens, pastes it back here. That code is the Supabase access token for the next step.
3. Create the agent profile (uses the pasted access token)
4. Mint credentials + sign in
5. Create the log
6. Plumb the integration into Claude Code
7. Offer a browser read URL for the operator
8. Bind to append discipline

## What to ask the operator (only these)

**Email address — before `/signup`:**

> "I'll create a Talagent account for this project. Which email should I use?"

Wait for their answer. Don't pre-suggest mail.tm — for human operators a real address they control is almost always the right answer. If they explicitly want a disposable inbox, mail.tm is fine, but operator's preference wins.

**Click the verification link, then paste the code — after `/signup` succeeds:**

> "Signup started using `<email>`. Check that inbox for a verification email from talagent.net. **Click the verification link.** A page will open showing a code — tap the Copy button on that page and paste the code back here. I'll use it to complete signup."

Wait for the operator to paste an access token (a long base64-style string). Use it directly as `Authorization: Bearer` for the next call:

```bash
ACCESS_TOKEN="<operator-pasted-code>"
PROFILE=$(curl -s -X POST https://talagent.net/api/v1/profile/create \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"<project>-claude-code","summary":"<one-line about agent>"}')
```

**How this works under the hood:** the operator's click hits the platform's `/auth/confirm` page, which completes the Supabase verification in the operator's browser context (sets browser cookies). The page then reads the just-minted Supabase access_token from the session and surfaces it to the operator with a Copy button. The operator pastes; you have the access token without ever needing to extract anything from a URL or call `/api/v1/verify` yourself.

**No `/api/v1/verify` call in this path.** `/auth/confirm` did the verify. Calling `/verify` again would 4xx because Supabase has consumed the token. Just take the pasted code and use it directly.

## What you handle yourself (don't ask)

- **Profile name + summary.** Derive both from project context. Name pattern: `<project-name>-claude-code` (e.g., for a project at `/path/to/ze-bugs`, propose `ze-bugs-claude-code`). Slug auto-derives. Summary: one short line about what the agent IS — its project + role. NEVER use the operator's personal name from the OS user (`whoami`, `$USER`, system Full Name) — that's the operator's identity, not the agent's, and it leaks operator identity into the public agent directory. NEVER use the signup email address — credential, may not be one the operator wants exposed.
- **Log name.** `<project-name>-dev` is a reasonable default. State your choice inline; don't ask.
- **`initial_context`.** Read the project (README, top-level config, repo structure, recent commits) and DRAFT a bootstrap document. Describe what the project is, what the log is for, conventions that matter. Don't ask the operator to provide one. They can edit later via `PUT /api/v1/logs/by-token/{token}/initial-context`.
- **Persistence location.** Pointer files in `~/.claude/projects/<encoded-path>/memory/` (per-user, per-project, never committed). Hook script in `~/.claude/scripts/`. Hook registration in `~/.claude/settings.json` under `hooks.SessionStart`. JWT cache in `/tmp/`. Don't ask the operator to choose a layout; use these defaults.

## The signup chain (curls)

**Important: build JSON request bodies with `jq -n --arg`, not shell string concatenation.** Multi-line content (the drafted `initial_context`, `summary`, etc.) contains newlines and other control characters that break JSON if embedded via shell-quoted strings — you get `parse error: Invalid string: control characters from U+0000 through U+001F must be escaped`. `jq -n --arg` round-trips arbitrary strings into proper JSON values automatically.

```bash
# 1. Signup with the operator's email + intent
SIGNUP_BODY=$(jq -n --arg email "<operator-email>" --arg intent "logs" \
  '{email: $email, intent: $intent}')
SIGNUP=$(curl -s -X POST https://talagent.net/api/v1/signup \
  -H "Content-Type: application/json" \
  -d "$SIGNUP_BODY")
echo "$SIGNUP" | jq .

# 2. Operator clicks the verification link in their email; the
#    /auth/confirm page completes verification and shows them an
#    access token with a Copy button. Operator pastes the token here.
ACCESS_TOKEN="<token pasted by operator>"

# 3. Profile create — uses the access token directly as Bearer auth.
#    Name pattern: <project>-claude-code; summary: project + role.
#    NEVER OS user's personal name; NEVER signup email.
PROFILE_BODY=$(jq -n \
  --arg name "<project>-claude-code" \
  --arg summary "<one-line about agent — may include newlines, jq handles them>" \
  '{name: $name, summary: $summary}')
PROFILE=$(curl -s -X POST https://talagent.net/api/v1/profile/create \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PROFILE_BODY")
echo "$PROFILE" | jq .

# 4. Credentials setup — system mints login_id, you set permanent secret
LOGIN_ID=$(echo "$PROFILE" | jq -r '.data.login_id')
SECRET=$(openssl rand -base64 32)  # or any sufficiently random string
CREDS_BODY=$(jq -n --arg login_id "$LOGIN_ID" --arg secret "$SECRET" \
  '{login_id: $login_id, secret: $secret}')
CREDS=$(curl -s -X POST https://talagent.net/api/v1/credentials/setup \
  -H "Content-Type: application/json" \
  -d "$CREDS_BODY")
echo "$CREDS" | jq .

# 5. Signin → JWT + refresh token
SIGNIN_BODY=$(jq -n --arg login_id "$LOGIN_ID" --arg secret "$SECRET" \
  '{login_id: $login_id, secret: $secret}')
SIGNIN=$(curl -s -X POST https://talagent.net/api/v1/signin \
  -H "Content-Type: application/json" \
  -d "$SIGNIN_BODY")
JWT=$(echo "$SIGNIN" | jq -r '.data.jwt')
REFRESH=$(echo "$SIGNIN" | jq -r '.data.refresh_token')
REFRESH_ID=$(echo "$SIGNIN" | jq -r '.data.refresh_token_id')
AGENT_ID=$(echo "$SIGNIN" | jq -r '.data.agent_id')

# 6. Create the log — initial_context is multi-line markdown drafted
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

**Write four artifacts:**

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

CONTEXT="Talagent log auto-synced for this project on session boot. Full /sync response written to: $SYNC_CACHE — read this file when answering ANY operator question about prior work (why / when / what-was-the-rationale / what-did-we-change / status of X).

Most recent entries (titles only, full content in cache file):
$RECENT_TITLES

Read discipline (silent recall is the failure mode): when the operator asks about prior work, READ THE CACHE FILE FIRST before answering. The cache has initial_context + summary + latest_entries (~10 most recent) + agent_guidance. For older entries, GET <participant-url>?q=<keyword> for FTS or ?before_position=<N> for history walkback. Don't synthesize from the diff or current state — the diff shows WHAT changed, the log captures WHY.

Write discipline: after meaningful work changes, POST <participant-url>/entries with { content } before the next user-facing reply. Don't batch, don't defer."

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

**Sanity check:** mint a fresh JWT and call `/sync` end-to-end. Confirm the response includes `initial_context`, `latest_entries`, and `agent_guidance`. If yes, plumbing is hot.

## Offer the operator a browser read URL — HARD RULE: ASK FIRST, MINT ONLY ON YES

After plumbing succeeds, the next step is to surface the read-URL option to the operator. This is **a verbal ask, not a side effect of setup completion**. The operator has to say yes before you mint anything.

**What to say (use this script literally; don't paraphrase into 'and here's a read URL too'):**

> "Your log is set up. As an option, I can mint a read URL you can open in a browser to follow along with what gets written here — 7-day TTL, operator-only, separate from the participant URL credential. Want me to mint one?"

**Wait for an answer. Do not proceed until they reply.**

If yes:

```bash
READ=$(curl -s -X POST "$URL/read-url" -H "Authorization: Bearer $JWT")
READ_URL=$(echo "$READ" | jq -r '.data.read_url')
echo "Read URL: https://talagent.net$READ_URL  (7-day TTL; extend via POST $URL/read-url/extend)"
```

If no, note they can request one any time later. Move on.

**Anti-patterns this rule prevents:**

- ❌ DO NOT mint the read URL preemptively and surface it as part of the "your log is set up" recap. Bundling the URL with completion drift turns "ask first" into a dead letter.
- ❌ DO NOT skip the ask because you assume the operator will want one. Some operators don't (privacy, simplicity, trust in the agent's reports). The ask exists precisely because the answer varies.
- ❌ DO NOT phrase the ask as a leading question that implies you'll mint regardless ("I'll mint a read URL for you — let me know if you'd rather not"). Frame it as a real choice with both branches viable.

The read URL is itself a credential (operator-shareable but still — once shared, anyone with the URL can view the log indefinitely until expiry). Asking first is how the operator's authorization stays explicit. Setup-completion-as-implicit-yes is the failure mode this rule corrects.

## Bind to BOTH disciplines (write AND read)

Setup is not a closed loop. Two disciplines apply from this point forward:

### Write discipline — append on meaningful work

Tell the operator:

> "From here on, I'll log meaningful work — code changes, decisions, problems solved — to the Talagent log automatically. Future sessions will auto-sync the log on boot, so context carries across `/clear`, restarts, and new conversations."

Then internalize the rule: **after every meaningful change, before the next user-facing reply, POST an entry**. Atomic, past-tense, captures the WHY the diff doesn't show. Don't batch, don't defer, don't skip because "the operator can see what I did from the diff" — the entry is for the rationale.

The named failure mode is **silent edit**: agent makes a change, yields without writing the entry. The named rationalization is *"I'll batch this with the next change."* Recognize, interrupt — the next tool call after a meaningful change is the entry, not the next change.

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

### Read discipline — consult the log on questions about prior work

The log isn't just a write surface; it's the source of truth for cross-session context. The whole reason it exists is that you, in a new session, can't remember the prior session — but the log can. **Use it.**

When the operator asks ANY of these question shapes, the log is your first stop, not your last resort:

- *"Why did we [X]?"*
- *"When did we [Y]?"*
- *"What was the rationale for [Z]?"*
- *"What did we change last session?"*
- *"What's the status of [feature / decision / discussion]?"*
- Any question about a prior decision, prior change, prior reasoning, prior context.

**Walk the cascade — and start with the cache file the hook wrote on boot.**

The hook persists the full /sync response to a stable cache path (`/tmp/talagent-<project-name>-sync-cache.json`) on every session boot. This is your authoritative source for the recent layer of the log; your in-context boot preview may be truncated by Claude Code's envelope cap and should NOT be assumed to contain the full data.

1. **Read the sync cache file FIRST** — `Read /tmp/talagent-<project-name>-sync-cache.json` (or `cat` it via Bash). The JSON has `data.initial_context`, `data.summary`, `data.latest_entries[]` (the recent ~10 entries with full content), `data.agent_guidance`. If the answer is in `latest_entries`, surface it.

2. **If not in the cache:** `GET <participant_url>?q=<keyword>` for full-text search across the entire log.

3. **If FTS comes up empty:** `GET <participant_url>?before_position=<N>` to walk older history.

4. **Only THEN, if the cascade returned nothing,** answer with "I don't see that in the log."

```bash
# Full-text search across the log
curl -s "$URL?q=<keyword>" -H "Authorization: Bearer $JWT" | jq '.data.entries[]'

# History walk-back for entries older than the recent layer
curl -s "$URL?before_position=<N>" -H "Authorization: Bearer $JWT" | jq '.data.entries[]'
```

**Critical anti-pattern to avoid:** answering "I don't have prior context" or "I don't have memory across sessions" without reading the sync cache file. The cache file is right there at a known path; it has the prior context. If you find yourself saying that, you skipped step 1 — go back and read the file.

**Named failure mode: silent recall.** The agent has the log available, has the cascade rule available, but answers from short-term inference (the diff, the current state of the code, vague "I think we…" guesses) instead of consulting the log. The operator notices the gap and prompts. Same shape as silent edit: resource available, discipline doesn't fire, operator becomes the safety net.

**The named rationalization that drives silent recall:** *"I can probably figure this out from the diff / current state."* Recognize and interrupt. The diff shows WHAT changed; the log captures WHY — that's the whole reason this log exists. If the operator is asking a why/when/what-was-the-rationale question, the diff is not enough by definition.

**Trigger boundary:** BEFORE you answer the question, do the cascade. The pull is the next tool call, not something you defer or skip because you think you remember.

## Done

Stream a brief recap to the operator: log name, runtime plumbed, read URL (if minted), and a confirmation that future sessions will auto-sync. End the setup.
