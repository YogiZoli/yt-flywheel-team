# Shared Conventions — YT Flywheel Orchestrators
Single source of truth for status artifacts, gate formats, context block schema,
global commands, resume logic, and error handling. Both real orchestrators
(playlist-strategy-orchestrator, video-production-orchestrator) read this file at startup:
`references/conventions.md` at the plugin root (relative to a grouped skill:
`../../../references/conventions.md`). `strategic-launch-orchestrator` is a pointer and needs none of this.

**Boundaries:** the Strategy and Execution layers are separated by the Video Brief Card; the
T1/T2/T3 teams are separated by `playlist_brief.json` / `optin_registry.json`. Both contracts
live in `references/layer-interface.md` — read it too.

---

## 1. CONTEXT BLOCK SCHEMA (orchestrator → agent handoff)

The orchestrator reads Drive ONCE at Step 0, then passes this block to EVERY agent it invokes.
Agents that receive a CONTEXT BLOCK MUST skip their own Pre-Flight Drive reads for the fields provided.
Agents invoked standalone (no context block) run their own Pre-Flight as written in their SKILL.md.

```
=== CONTEXT BLOCK v2 ===
CHANNEL_NAME: [name]
CHANNEL_LANG: [HU | EN | ... — from _Channel_Pin.txt; drives ALL public-facing copy language]
CHANNEL_FOLDER_ID: [Drive folder ID]
SUBFOLDER_IDS: 00=[id] | 01=[id] | 02=[id] | 03=[id] | 04=[id] | 05=[id] | 06=[id] | 07=[id] | 08=[id] | 09=[id] | Captions=[id]
ACTIVE_OPTIN_URL: [URL or null]
BRIEF_ID: [brief_id of the Video Brief Card this run executes | n/a for Layer A / ad-hoc]
VIDEO_TYPE: [avatar_local | avatar_native | screen_recording | talking_head | faceless_generated | n/a]   ← sourced from the Brief Card, priority order most→least used
PLAYLIST: [SeriesName] | EP.[X] of [Y] | Arc: [Opening/Middle/Closing]  ← sourced from the Brief Card
LEAD_MAGNET: [name] | [hosted URL or pending]
AVATAR_SUMMARY: [2-3 sentence extract from Foundation doc]
CONFIRMED_TITLE: [title or n/a]
CONFIRMED_TOPIC: [topic or n/a — for a video run this is the Brief Card topic]
=== END CONTEXT BLOCK ===
```

Rules:
- Orchestrator resolves ALL subfolder IDs in one scan at Step 0 (one `search_files(parentId=channel)` call).
- Fields not yet known at handoff time are passed as `n/a` — the agent may read only those from Drive.
- Agents never re-read `_Active_Channel`, the Foundation doc, or subfolder IDs when the block provides them.
- **When `BRIEF_ID` is set**, the named Video Brief Card in `01_Strategy/` is read-only truth for
  topic / primary_keyword / angle / video_type / lead magnet / playlist position. An agent that
  thinks one of those is wrong STOPS and reports — it does not edit the card. See `layer-interface.md`.
- v1 blocks (no `BRIEF_ID`) are still accepted: an agent that receives one runs as before.

## 2. STATUS ARTIFACT — SHARED HTML BASE

Both orchestrators use this CSS/skeleton. Only the step rows and header differ.

```html
<!DOCTYPE html><html><head><meta charset="UTF-8"><style>
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;
background:#0f0f0f;color:#e0e0e0;padding:24px;max-width:540px;margin:0 auto}
h2{color:#fff;font-size:18px;margin-bottom:4px}
.ch{color:#888;font-size:13px;margin-bottom:6px}
.optin{color:#82B0F7;font-size:12px;margin-bottom:16px;word-break:break-all}
.step{display:flex;align-items:center;gap:12px;padding:10px 14px;
border-radius:8px;margin-bottom:6px;background:#1a1a1a}
.step.done{background:#0d2b1a;border-left:3px solid #22c55e}
.step.active{background:#15355F;border-left:3px solid #818cf8}
.step.waiting{background:#2b1a0d;border-left:3px solid #f59e0b}
.step.skipped{background:#1a1a1a;border-left:3px solid #555;opacity:.6}
.step.pending{opacity:.4;border-left:3px solid #333}
.icon{font-size:18px;width:24px;text-align:center}
.label{flex:1;font-size:14px;font-weight:500}
.status{font-size:12px;color:#666}
.step.done .status{color:#22c55e}.step.active .status{color:#818cf8}
.step.waiting .status{color:#f59e0b}
.gate{font-size:11px;color:#444;margin-left:36px;margin-bottom:8px;margin-top:-2px}
.urlbox{background:#0d2b1a;border:1px solid #22c55e;border-radius:8px;
padding:12px 16px;margin-top:16px;font-size:13px;display:none}
.urlbox.visible{display:block}
.urllabel{color:#22c55e;font-weight:600;margin-bottom:4px}
.urlval{color:#fff;word-break:break-all}
</style></head><body>
<h2><!-- HEADER: YT Pipeline | Strategic Launch --></h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --></div>
<!-- STEP ROWS: <div class="step STATE"><span class="icon">EMOJI</span>
<span class="label">LABEL</span><span class="status">STATUS_TEXT</span></div>
<div class="gate">▸ Gate N: description</div> -->
</body></html>
```

Step classes: `done` / `active` / `waiting` / `skipped` / `pending`
Status text: `"done"` / `"running..."` / `"waiting for approval"` / `"skipped"`
Create via `mcp__cowork__create_artifact` at Step 0; update via `mcp__cowork__update_artifact` after every step.

## 3. GATE BLOCK FORMAT

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE [N] — [Gate name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[instructions]
Approve → "approve" / "ok" / "go"
Change → "change: [what to change]"
Full regeneration → "redo"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Never proceed past a gate without an explicit operator response.

## 4. GLOBAL COMMANDS

| Command | Action |
|---------|--------|
| `"skip"` / `"skip S[N]"` | Skip current/named step, keep existing Drive output |
| `"redo"` / `"redo [N]"` | Re-run current/named step from scratch |
| `"hol tartunk"` | Show status artifact |
| `"what is left"` | List remaining steps |
| `"stop"` | Pause, note current position |
| `"manual [N]"` | Output manual instructions instead of automating |

## 5. RESUME LOGIC

On start, scan Drive for partial completion. Present found/missing per step, then:
```
Continue from the next missing step? → "yes"
Start over → "start over"
Re-run a given step → "redo [N]"
```
Skip already-completed steps by default. Always show what is being skipped.

## 6. ERROR HANDLING

1. Surface every error explicitly — never skip silently.
2. Present: `"[Step N] reported an error: [error]. Try again? → 'yes' / 'skip'"`
3. On retry: re-invoke the same agent skill from its start.
4. On skip: mark step SKIPPED (not DONE) in artifact, continue with existing Drive output.
5. Tool-specific fallbacks: GHL MCP fails → structured brief + manual GHL UI steps.
   Canva MCP fails → per-page design brief for manual execution.
   Chrome MCP unavailable → full HTML file + manual publish steps.
   Never block the pipeline on an optional tool failure.

## 7. CHANNEL PIN RESOLUTION (switch-free multi-channel)

Goal: the operator NEVER has to remember or switch the active channel. Resolution order —
first hit wins, run at Step 0a of every orchestrator and in every standalone agent Pre-Flight:

1. **Pin file in the connected working folder** (HIGHEST priority):
   look for `_Channel_Pin.txt` in the Cowork session's connected folder (and its root).
   Format, line 1: `[Channel Name] | [LANG] | [Channel Drive Folder ID]`
   Example: `VoxFlow | HU | 1AbC...xyz`
   If found → the session is HARD-BOUND to this channel:
   - All outputs go to this channel's Drive folder. All copy is written in [LANG].
   - `"switch to ..."` commands are REFUSED in this session with:
     "This working folder is pinned to the [Channel] channel. For another channel, open that channel's folder."
   - Do NOT read `_Active_Channel` at all.
2. **Explicit channel in the trigger message** (e.g. "new video @<channel>") → use it for this run.
3. **`_Active_Channel` pointer on Drive** (legacy fallback) → allowed, but the orchestrator MUST
   display a prominent channel badge (name + language) in the status artifact AND in the first
   gate message, so the operator always sees which channel is live.

Language rule: the pin's [LANG] (HU/EN/...) drives ALL public-facing output language
(scripts, titles, descriptions, CTAs, community posts) for that channel — no per-task language
questions, ever. Internal chat with the operator follows the operator's language as usual.

Per-channel setup (one-time, channel-manager does it on "create new channel"):
- Drive: `YT_Flywheel/[Channel]/` structure (as before)
- Local: `C:\Users\gdp\Documents\Claude\Projects\YouTube\[Channel]\` working folder
  containing `_Channel_Pin.txt` — the operator connects THIS folder in Cowork sessions
  dedicated to that channel.
