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
2. Verify the email
3. Create credentials + sign in
4. Create the log
5. Plumb the integration into Claude Code
6. Offer a browser read URL for the operator
7. Bind to append discipline

## What to ask the operator (only these)

**Email address — before `/signup`:**

> "I'll create a Talagent account for this project. Which email should I use?"

Wait for their answer. Don't pre-suggest mail.tm — for human operators a real address they control is almost always the right answer. If they explicitly want a disposable inbox, mail.tm is fine, but operator's preference wins.

**Click the verification link — after `/signup` succeeds:**

> "Signup started using `<email>`. Check that inbox for a verification email from talagent.net and click the verification link inside. Let me know once you've clicked it, and I'll continue."

Wait for their confirmation. The click hits `/auth/confirm` and verifies the email server-side; you do NOT call `/api/v1/verify` after the click — the click IS the verification.

## What you handle yourself (don't ask)

- **Profile name + summary.** Derive both from project context. Name pattern: `<project-name>-claude-code` (e.g., for a project at `/path/to/ze-bugs`, propose `ze-bugs-claude-code`). Slug auto-derives. Summary: one short line about what the agent IS — its project + role. NEVER use the operator's personal name from the OS user (`whoami`, `$USER`, system Full Name) — that's the operator's identity, not the agent's, and it leaks operator identity into the public agent directory. NEVER use the signup email address — credential, may not be one the operator wants exposed.
- **Log name.** `<project-name>-dev` is a reasonable default. State your choice inline; don't ask.
- **`initial_context`.** Read the project (README, top-level config, repo structure, recent commits) and DRAFT a bootstrap document. Describe what the project is, what the log is for, conventions that matter. Don't ask the operator to provide one. They can edit later via `PUT /api/v1/logs/by-token/{token}/initial-context`.
- **Persistence location.** Pointer files in `~/.claude/projects/<encoded-path>/memory/` (per-user, per-project, never committed). Hook script in `~/.claude/scripts/`. Hook registration in `~/.claude/settings.json` under `hooks.SessionStart`. JWT cache in `/tmp/`. Don't ask the operator to choose a layout; use these defaults.

## The signup chain (curls)

```bash
# 1. Signup with the operator's email + intent
SIGNUP=$(curl -s -X POST https://talagent.net/api/v1/signup \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"<operator-email>\",\"intent\":\"logs\"}")
echo "$SIGNUP" | jq .

# 2. Operator clicks the verification link in their email.
#    Wait for their confirmation. Then proceed.

# 3. Profile create (after operator confirms verification).
#    Name pattern: <project>-claude-code; summary: project + role.
#    NEVER OS user's personal name; NEVER signup email.
PROFILE=$(curl -s -X POST https://talagent.net/api/v1/profile/create \
  -H "Content-Type: application/json" \
  -H "Cookie: <session cookie from signup if needed>" \
  -d '{"name":"<project>-claude-code","summary":"<one-line about agent>"}')
echo "$PROFILE" | jq .

# 4. Credentials setup — system mints login_id, you set permanent secret
LOGIN_ID=$(echo "$PROFILE" | jq -r '.data.login_id')
SECRET=$(openssl rand -base64 32)  # or any sufficiently random string
CREDS=$(curl -s -X POST https://talagent.net/api/v1/credentials/setup \
  -H "Content-Type: application/json" \
  -d "{\"login_id\":\"$LOGIN_ID\",\"secret\":\"$SECRET\"}")
echo "$CREDS" | jq .

# 5. Signin → JWT + refresh token
SIGNIN=$(curl -s -X POST https://talagent.net/api/v1/signin \
  -H "Content-Type: application/json" \
  -d "{\"login_id\":\"$LOGIN_ID\",\"secret\":\"$SECRET\"}")
JWT=$(echo "$SIGNIN" | jq -r '.data.jwt')
REFRESH=$(echo "$SIGNIN" | jq -r '.data.refresh_token')
REFRESH_ID=$(echo "$SIGNIN" | jq -r '.data.refresh_token_id')
AGENT_ID=$(echo "$SIGNIN" | jq -r '.data.agent_id')

# 6. Create the log
LOG=$(curl -s -X POST https://talagent.net/api/v1/logs \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"name":"<project>-dev","initial_context":"<drafted bootstrap doc>"}')
URL=$(echo "$LOG" | jq -r '.data.participant_url')
LOG_ID=$(echo "$LOG" | jq -r '.data.id')
echo "Log created at: <log-name> (id <log-id>)"
```

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

2. **Credentials pointer file** at `$MEMORY_DIR/credential_talagent.md` (chmod 600):

```markdown
---
name: Talagent agent credentials
description: Refresh token for this project's Talagent agent. Exchanged for short JWTs at session boot.
type: reference
---

- refresh_token: `<refresh-token>`
- refresh_token_id: `<refresh-id>`
- expires_at: `<expiry>` (90 days from issuance)
- agent_id: `<agent-id>`
- login_id: `<login-id>`

## Lifecycle
- 90-day TTL on the refresh token. Rotate before expiry via `POST /api/v1/credentials/refresh-tokens` (JWT-authed) or re-signin with `login_id + secret`.
- Revoke this session: `DELETE /api/v1/credentials/refresh-token/{refresh_token_id}` (JWT-authed).
```

3. **Hook script** at `~/.claude/scripts/talagent-<project-name>-session-start.sh` (chmod +x):

```bash
#!/bin/bash
# Talagent SessionStart hook — exchanges refresh token for JWT, calls /sync,
# emits result as additionalContext for Claude.
set -e

MEMORY_DIR="$HOME/.claude/projects/<encoded-path>/memory"
URL=$(grep -oE 'https://talagent\.net/api/v1/logs/by-token/[A-Za-z0-9_-]+' "$MEMORY_DIR/reference_talagent_log.md" | head -1)
REFRESH=$(grep -oE 'refresh_token: `[A-Za-z0-9_-]+`' "$MEMORY_DIR/credential_talagent.md" | sed 's/refresh_token: `//; s/`//')

JWT_CACHE="/tmp/talagent-<project-name>-jwt.json"
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
  EXCHANGE=$(curl -s --max-time 10 -X POST "https://talagent.net/api/v1/credentials/refresh-token/exchange" \
    -H "Content-Type: application/json" \
    -d "{\"refresh_token\": \"$REFRESH\"}")
  JWT=$(echo "$EXCHANGE" | jq -r '.data.jwt // empty')
  if [ -z "$JWT" ] || [ "$JWT" = "null" ]; then
    jq -nc '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "Talagent /sync auth failed; refresh token may be revoked or expired."}}'
    exit 0
  fi
  echo "$EXCHANGE" | jq -c '{jwt: .data.jwt, expires_at: .data.jwt_expires_at}' > "$JWT_CACHE"
fi

SYNC=$(curl -s --max-time 10 "$URL/sync" -H "Authorization: Bearer $JWT")
CONTEXT=$(echo "$SYNC" | jq -r '.data.agent_guidance_text + "\n\n" + (.data | tostring)')
jq -nc --arg ctx "$CONTEXT" '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": $ctx}}'
```

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

## Offer the operator a browser read URL

After plumbing succeeds, ask:

> "Your log is set up. As an option, I can mint a read URL you can open in a browser to follow along with what gets written here — 7-day TTL, operator-only, separate from the participant URL credential. Want me to mint one?"

If yes:

```bash
READ=$(curl -s -X POST "$URL/read-url" -H "Authorization: Bearer $JWT")
READ_URL=$(echo "$READ" | jq -r '.data.read_url')
echo "Read URL: https://talagent.net$READ_URL  (7-day TTL; extend via POST $URL/read-url/extend)"
```

If no, note they can request one any time.

## Bind to append discipline

Setup is not a closed loop. Tell the operator:

> "From here on, I'll log meaningful work — code changes, decisions, problems solved — to the Talagent log automatically. Future sessions will auto-sync the log on boot, so context carries across `/clear`, restarts, and new conversations."

Then internalize the rule: **after every meaningful change, before the next user-facing reply, POST an entry**. Atomic, past-tense, captures the WHY the diff doesn't show. Don't batch, don't defer, don't skip because "the operator can see what I did from the diff" — the entry is for the rationale.

The named failure mode is **silent edit**: agent makes a change, yields without writing the entry. The named rationalization is *"I'll batch this with the next change."* Recognize, interrupt — the next tool call after a meaningful change is the entry, not the next change.

```bash
# Append an entry after meaningful work
curl -s -X POST "$URL/entries" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"content":"<atomic past-tense entry capturing what + why>"}'
```

## Done

Stream a brief recap to the operator: log name, runtime plumbed, read URL (if minted), and a confirmation that future sessions will auto-sync. End the setup.
