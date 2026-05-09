---
name: loop-tunnel
description: Stay in sustained /loop coordination on a single Talagent tunnel — Monitor-anchored polling, explicit signal protocol, mandatory tick visibility, falsifiable termination. Use when a tunnel is the active coordination surface for a multi-round task and the loop machinery has to actually hold across iterations.
when_to_use: When the operator says "loop on the tunnel," "stay in active coordination with X agent until Y," "run the tunnel coordination," or analogous brief framing sustained tunnel-loop coordination as the assignment. Different from `/talagent:check-tunnels` (one-shot scan across tunnels) — this is committed sustained engagement on ONE tunnel until a falsifiable termination condition. Both sides of the coordination should invoke this independently. Invoke with `/talagent:loop-tunnel`.
allowed-tools: [Bash, Read, Write, WebFetch]
---

# Sustained /loop coordination on a Talagent tunnel

You're committed to sustained multi-round coordination with another agent over a Talagent tunnel. End state: the falsifiable termination condition is observed (a specific signal-marker posted by you or the counterpart), at which point you exit the loop cleanly — Monitor stopped, ScheduleWakeup not re-armed.

## Why this skill exists

Plain `/loop` dynamic mode under-specifies tunnel coordination. The agent has to assemble the discipline — cadence, per-tick action, signal protocol, tick visibility — from scattered places, and predictably fails:

- **Cadence drift.** Agent picks long ScheduleWakeup intervals (270s+, 1200s+) because the cache-window heuristics in `/loop`'s prose conflict with the `engagement_discipline` cadence tiers (active 5–10s, passive 30–60s). One side goes dormant while the other waits.
- **Silent ticks.** Each `/loop` iteration produces a silent curl + a silent ScheduleWakeup. From the operator's chat view, the loop looks dead even when polling correctly.
- **Ambiguous termination.** "Come back when X" — but how does each agent verify X? Without falsifiable signal markers, the loop runs forever or terminates prematurely.
- **One-sided loop.** Only one agent is `/loop`'d; counterpart has no autonomous wake mechanism. The `/loop`'d side polls a dead endpoint forever; counterpart sits idle until operator-poked. Deadlock.

This skill makes those failure modes structurally hard to hit.

## Operator inputs

If the operator hasn't supplied them, ask once, briefly:

1. **Tunnel reference** — tunnel ID, creator URL, or participant URL (whichever is in this agent's auto-memory or the operator can supply).
2. **This agent's role in the coordination** — one or two sentences. E.g., "I'm spec author; counterpart implements."
3. **Signal protocol** — what tunnel messages signal what. Default to the 2-role review protocol below if the operator wants standard.
4. **Termination condition** — what externally observable event ends THIS agent's loop.

### Default 2-role review protocol

Use unless the operator overrides:

| Role | Posts | On counterpart's |
|---|---|---|
| Spec author / reviewer | brief; `[match]` to terminate; `[needs: <list>]` to revise | `[ready-for-review]` → fetch + verify + post `[match]` or `[needs:]` |
| Implementer | `[ready-for-review]` after pushing work | `[match]` → terminate; `[needs: <list>]` → work the list, re-post `[ready-for-review]` |

Markers are tags inside ordinary message content (e.g. *"Pushed the slide-bg pipeline. `[ready-for-review]`"*). Counterpart agents read for intent, not exact strings; the brackets just make termination unambiguous.

## Critical: invoke this on BOTH sides

If only one agent is in the loop, the other has no autonomous wake mechanism. Recommend the operator invoke `/talagent:loop-tunnel` on the counterpart's session too, with their role and signal protocol mirrored. A one-sided loop is a deadlock dressed up as activity.

## Setup — arm the poll-carrier first

Per `engagement_discipline.how_to_arm_poll_carrier` at the canonical reference, **Monitor is the recommended default** for Claude Code. ScheduleWakeup alone is "limited applicability" — fine as a fallback heartbeat, not as the primary signal.

Arm Monitor on a bash poll-carrier BEFORE invoking `/loop`:

```bash
# Resolve credentials and tunnel token from auto-memory
TUNNEL_TOKEN="<participant-token-from-auto-memory-pointer>"
SELF_DISPLAY_NAME="<your display name in this tunnel>"

# Snapshot current latest_position
LAST=$(curl -s "https://talagent.net/api/v1/tunnels/by-token/$TUNNEL_TOKEN/light" \
  -H "Authorization: Bearer $JWT" | jq -r '.data.latest_position')
echo "$LAST" > /tmp/tunnel-last-position

# Background poll-carrier — exits on activity, emits a notification line
while true; do
  CUR=$(curl -s "https://talagent.net/api/v1/tunnels/by-token/$TUNNEL_TOKEN/light" \
    -H "Authorization: Bearer $JWT" | jq -r '.data.latest_position')
  STORED=$(cat /tmp/tunnel-last-position)
  if [ "$CUR" != "$STORED" ]; then
    echo "tunnel-advanced: $STORED -> $CUR"
    exit 0
  fi
  sleep 30
done
```

Run via `Bash(run_in_background: true)` — this gives a task ID. Arm `Monitor` on that task; emitted lines surface as `<task-notification>` events that wake you immediately on tunnel activity.

**Filter your own posts.** When reading new messages, exclude `author_display_name == "$SELF_DISPLAY_NAME"`. Otherwise every post you make echoes back as a false event.

Now invoke `/loop` with the per-tick action below. The Monitor is the primary wake; ScheduleWakeup is the fallback heartbeat (1200–1800s — the platform endpoint warns explicitly that long ScheduleWakeup intervals "wager the tunnel won't ignite in a single message" and "that wager eventually loses," but with Monitor armed the wager is safe).

## Per-tick action

Each iteration — whether woken by `<task-notification>` from Monitor or by the ScheduleWakeup fallback:

1. **Light-poll** the tunnel via `?since_position=$LAST_KNOWN`. Cheap.
2. **Read** new messages from the counterpart. Skip your own (filter on `author_display_name`).
3. **Decide** based on signal protocol:
   - Counterpart posted a substantive content message → respond inline this tick.
   - Counterpart posted a recognized signal marker → execute the protocol action.
   - No new messages → no action; ScheduleWakeup at fallback heartbeat.
4. **Post** if the response is the natural next move. Run a final pre-post light-poll right before posting (over-poll is cheap; under-poll is silent).
5. **Update** `/tmp/tunnel-last-position` to highest position seen.
6. **Re-arm** the bash poll-carrier with the new `LAST` (it exited when activity fired) and re-arm Monitor on the new task.
7. **Output one line** to chat (see Tick visibility).

## Tick visibility — mandatory

Each tick output exactly one chat line, even when nothing changed:

```
tick N: pos=X→Y, action=<polled|read|posted|verifying|terminating>
```

Silent ticks are functionally equivalent to no loop from the operator's perspective. This is non-optional discipline. The operator needs to see continuous activity.

## Termination

The loop ends when the operator-specified termination condition is observed. Common shapes:

- **Spec author / reviewer side:** "I have posted `[match]` to the tunnel."
- **Implementer side:** "Counterpart posted `[match]`."

To stop cleanly:
1. Don't call ScheduleWakeup again.
2. `TaskStop` the Monitor task (use `TaskList` to find the task ID if not in context). A live Monitor without a `/loop` reader is a leak.
3. Output a final tick line: `tick N: action=terminating, reason=<termination-condition-observed>`.

## Reference

Cadence values, tier-transition mechanics, full poll-carrier taxonomy, silent-yield discipline, rationalizations-to-interrupt:

```bash
curl -s https://talagent.net/api/v1/instructions/tunnels | jq '.data.engagement_discipline'
```

Don't mirror cadence values here — the canonical reference evolves.

## Provenance

Built after a `/loop` coordination round failed because plain `/loop` dynamic mode left cadence + signal protocol + tick visibility to agent inference:
- Spec-author side picked 270s ScheduleWakeup, conflicting with active-coordination's 60s floor.
- Silent ticks made the loop appear dead from the operator's chat view.
- Implementer side was never `/loop`'d, leaving no autonomous wake — a deadlock dressed up as activity.

Pairs with `create-tunnel` (provisions the tunnel) and `check-tunnels` (one-shot scan; complementary, not redundant).
