# Shared Conventions — YT Flywheel Orchestrators
Single source of truth for status artifacts, gate formats, context block schema,
global commands, resume logic, and error handling. Both orchestrators
(video-production-orchestrator, strategic-launch-orchestrator) read this file at startup:
`references/conventions.md` at the plugin root (relative to the skill: `../../references/conventions.md`).

---

## 1. CONTEXT BLOCK SCHEMA (orchestrator → agent handoff)

The orchestrator reads Drive ONCE at Step 0, then passes this block to EVERY agent it invokes.
Agents that receive a CONTEXT BLOCK MUST skip their own Pre-Flight Drive reads for the fields provided.
Agents invoked standalone (no context block) run their own Pre-Flight as written in their SKILL.md.

```
=== CONTEXT BLOCK v1 ===
CHANNEL_NAME: [name]
CHANNEL_FOLDER_ID: [Drive folder ID]
SUBFOLDER_IDS: 00=[id] | 01=[id] | 02=[id] | 03=[id] | 04=[id] | 05=[id] | 06=[id] | 07=[id] | 08=[id] | 09=[id] | Captions=[id]
ACTIVE_OPTIN_URL: [URL or null]
VIDEO_TYPE: [talking_head | heygen_avatar | screen_recording | n/a]
PLAYLIST: [SeriesName] | EP.[X] of [Y] | Arc: [Opening/Middle/Closing]
LEAD_MAGNET: [name] | [hosted URL or pending]
AVATAR_SUMMARY: [2-3 sentence extract from Foundation doc]
CONFIRMED_TITLE: [title or n/a]
CONFIRMED_TOPIC: [topic or n/a]
=== END CONTEXT BLOCK ===
```

Rules:
- Orchestrator resolves ALL subfolder IDs in one scan at Step 0 (one `search_files(parentId=channel)` call).
- Fields not yet known at handoff time are passed as `n/a` — the agent may read only those from Drive.
- Agents never re-read `_Active_Channel`, the Foundation doc, or subfolder IDs when the block provides them.

## 2. STATUS ARTIFACT — SHARED HTML BASE

Both orchestrators use this CSS/skeleton. Only the step rows and header differ.

```html
<!DOCTYPE html><html><head><meta charset="UTF-8"><style>
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;
background:#0f0f0f;color:#e0e0e0;padding:24px;max-width:540px;margin:0 auto}
h2{color:#fff;font-size:18px;margin-bottom:4px}
.ch{color:#888;font-size:13px;margin-bottom:6px}
.optin{color:#00C4FF;font-size:12px;margin-bottom:16px;word-break:break-all}
.step{display:flex;align-items:center;gap:12px;padding:10px 14px;
border-radius:8px;margin-bottom:6px;background:#1a1a1a}
.step.done{background:#0d2b1a;border-left:3px solid #22c55e}
.step.active{background:#1a1a2e;border-left:3px solid #818cf8}
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
<h2><!-- HEADER: 🎬 YT Pipeline | 🚀 Strategic Launch --></h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --></div>
<!-- STEP ROWS: <div class="step STATE"><span class="icon">EMOJI</span>
<span class="label">LABEL</span><span class="status">STATUS_TEXT</span></div>
<div class="gate">▸ Gate N: description</div> -->
</body></html>
```

Step classes: `done` / `active` / `waiting` / `skipped` / `pending`
Status text: `"✓ kész"` / `"→ fut..."` / `"⏸ jóváhagyásra vár"` / `"—"`
Create via `mcp__cowork__create_artifact` at Step 0; update via `mcp__cowork__update_artifact` after every step.

## 3. GATE BLOCK FORMAT

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE [N] — [Gate name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[instructions]
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Módosítás → "módosítsd: [mi változzon]"
Teljes újragenerálás → "újra"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Never proceed past a gate without an explicit operator response.

## 4. GLOBAL COMMANDS

| Command | Action |
|---------|--------|
| `"skip"` / `"skip S[N]"` | Skip current/named step, keep existing Drive output |
| `"újra"` / `"újra [N]"` | Re-run current/named step from scratch |
| `"hol tartunk"` | Show status artifact |
| `"mi van még"` | List remaining steps |
| `"állj meg"` | Pause, note current position |
| `"manuális [N]"` | Output manual instructions instead of automating |

## 5. RESUME LOGIC

On start, scan Drive for partial completion. Present found/missing per step, then:
```
Folytatás a következő hiányzó lépéstől? → "igen"
Elölről kezdés → "újrakezdem"
Adott lépés újrafuttatása → "újra [N]"
```
Skip already-completed steps by default. Always show what is being skipped.

## 6. ERROR HANDLING

1. Surface every error explicitly — never skip silently.
2. Present: `"[Step N] hibát jelzett: [error]. Próbáljuk újra? → 'igen' / 'átugrás'"`
3. On retry: re-invoke the same agent skill from its start.
4. On skip: mark step SKIPPED (not DONE) in artifact, continue with existing Drive output.
5. Tool-specific fallbacks: GHL MCP fails → structured brief + manual GHL UI steps.
   Canva MCP fails → per-page design brief for manual execution.
   Chrome MCP unavailable → full HTML file + manual publish steps.
   Never block the pipeline on an optional tool failure.
