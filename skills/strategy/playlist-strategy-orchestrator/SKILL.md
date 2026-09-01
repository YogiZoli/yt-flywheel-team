---
name: playlist-strategy-orchestrator
description: LAYER A controller — designs a whole YouTube playlist and emits the per-video Video Brief Cards that the execution layer consumes, plus the playlist_brief.json handoff for the T2 funnel team. Runs once per playlist, plus re-plan runs after an analytics review. Triggers ONLY when the user says "playlist stratégia", "playlist strategy", "tervezd meg a playlistet", "plan the playlist", "playlist koncepció", "playlist újratervezés", "re-plan the playlist", or "playlist brief". Chain: Agent 01 (Playlist Mode) → Agent 02 (Playlist Keyword Mode) → emit PlaylistStrategy doc + one Video Brief Card per planned video + playlist_brief.json, then STOP. Does NOT build the funnel (T2: CMO orchestrator / email-list-building reads playlist_brief.json on its own) and does NOT produce any single video (video-production-orchestrator).
---

# Playlist Strategy Orchestrator (Layer A)
**Role:** Channel/playlist-level decision controller. Decides the whole playlist ONCE, then
emits the interface artifacts the execution layer runs on.
**Never does:** funnel building, lead magnet content, a single video's SEO title / script /
thumbnail / publish. Those are Layer B and the email-list-building plugin.

**Output = the interface (see `references/layer-interface.md`):**
- one `PlaylistStrategy_[SeriesName]` doc (umbrella concept + ordered sequence)
- one `Brief_[SeriesSlug]_EP[NN]_[slug]` Video Brief Card per planned video, `status: planned`

The Video Production Orchestrator later picks up the lowest-EP `planned` card and runs it.
Zero strategy thinking happens at video time.

---

## SHARED CONVENTIONS — READ FIRST
At startup read BOTH:
- `../../../references/conventions.md` — CONTEXT BLOCK schema (§1), status artifact HTML base (§2),
  gate format (§3), global commands (§4), resume logic (§5), error handling (§6), channel pin (§7).
- `../../../references/layer-interface.md` — the Brief Card v1 and PlaylistStrategy v1 schemas,
  the "who writes which field" table, the feedback loop.

Build the CONTEXT BLOCK at Step 0 and pass it to every skill you invoke. `BRIEF_ID: n/a`
for this orchestrator (Layer A operates on the whole playlist, not one card).

---

## MODES

| Mode | Trigger | What it does |
|---|---|---|
| **Fresh** | "playlist stratégia", "tervezd meg a playlistet" | Full run: concept → keyword map → doc + all Brief Cards |
| **Re-plan** | "playlist újratervezés" | Reads the latest `06_Analytics_Reviews/` doc, rewrites/reorders only the `planned` Brief Cards from the current EP onward, bumps `last_replanned`. Never touches `published` cards. |

---

## STEP 0 — STARTUP

### 0a. Resolve channel — CHANNEL PIN FIRST (conventions.md §7)
1. **Pin file** `_Channel_Pin.txt` in the connected folder → `[Channel] | [LANG] | [Folder ID]` →
   session HARD-BOUND: use these values, write all public copy in [LANG], REFUSE "switch", do
   NOT read `_Active_Channel`.
2. **Explicit channel in trigger** ("playlist stratégia @VoxFlow") → resolve that folder ID.
3. **Legacy fallback:** `search_files(query: "title = '_Active_Channel'")` → parse line 1.
   Show a prominent channel badge (name + language) in the artifact and at Gate 1.
4. Nothing resolves → invoke `channel-manager` first.

### 0b. Read Foundation
`search_files(query: "title = '00_Foundation' and parentId = '[channelFolderID]'")` → subfolder →
read most recent doc. Extract: Avatar Card (viewer identity + transformation), Hero Story, niche.

### 0c. Read latest analytics (input to strategy — always, both modes)
`search_files(query: "title = '06_Analytics_Reviews' and parentId = '[channelFolderID]'")` →
subfolder → read most recent review if any. Extract the **CONTENT DIRECTION UPDATE** section
and any **BREAKTHROUGH VIDEO FLAGGED**. In Re-plan mode this is mandatory; if there is no
review, tell the operator and ask whether to proceed on judgement alone.

### 0d. Check existing strategy
`search_files(query: "title contains 'PlaylistStrategy' and parentId = '[01_Strategy subfolderID]'")`.
- **Fresh mode + strategy exists** → show it, ask: "Van már playlist stratégia: [name].
  Új playlistet indítunk, vagy ezt tervezzük újra? → 'új' / 'újratervezés'".
- **Re-plan mode + no strategy** → "Nincs mit újratervezni. Előbb: 'playlist stratégia'." Stop.

### 0e. Build CONTEXT BLOCK + create Status Artifact
Resolve all subfolder IDs in one `search_files(parentId=channel)` call. Assemble the CONTEXT
BLOCK per conventions.md §1 with `BRIEF_ID: n/a`. Create the status artifact via
`mcp__cowork__create_artifact`, update after every step.

## STATUS ARTIFACT (HTML)
Shared HTML base from `conventions.md` §2. Header: `🧭 Playlist Strategy`. Step rows:

```html
<h2>🧭 Playlist Strategy</h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --> · <!-- MODE: Fresh | Re-plan --></div>
<div class="step <!-- S1 -->"><span class="icon">🧠</span>
<span class="label">S1 — Playlist Concept + Sequence (Agent 01)</span><span class="status"><!-- S1S --></span></div>
<div class="gate">▸ Gate 1: Approve concept + video sequence</div>
<div class="step <!-- S2 -->"><span class="icon">🔍</span>
<span class="label">S2 — Per-Video Keyword Map (Agent 02)</span><span class="status"><!-- S2S --></span></div>
<div class="step <!-- S3 -->"><span class="icon">🗂️</span>
<span class="label">S3 — Emit PlaylistStrategy doc + Brief Cards</span><span class="status"><!-- S3S --></span></div>
```

---

## STEP S1 — Agent 01: Playlist Concept + Sequence

Invoke `agent-01-channel-strategist` with the CONTEXT BLOCK and this brief:

> "Run in **Playlist Mode**. Do NOT pick a single next video. Design the whole playlist:
> - series name + series slug + audience (from the Avatar Card) + positioning vs the channel's
>   other playlists
> - ONE umbrella keyword the whole playlist ladders up to
> - the single transformation arc the playlist delivers
> - 8–12 videos in Evergreen-first order; per video: topic, type (Evergreen/Viral/Depth),
>   arc position (Opening EP1–2 / Middle / Closing), and `video_type`
>   (talking_head / heygen_avatar / screen_recording) chosen by content fit
> - `default_video_type` for the playlist + `target_type_mix` (guide 65/25/10)
> - the ONE lead magnet transformation this playlist's companion guide delivers + slot code
> - recommended CTA language, consistent across every video
> - if analytics input exists (Step 0c): fold the CONTENT DIRECTION UPDATE into the sequence"

Agent 01 writes the concept to a working note in `01_Strategy/`. Update artifact S1 → `waiting`.
Then:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 1 — Playlist koncepció + sorrend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Csatorna: [CHANNEL_NAME] ([CHANNEL_LANG])
Playlist: [series_name] · umbrella keyword: [kw] · [N] videó
Sorrend: EP01 [topic] … EP[N] [topic]

Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Módosítás → "módosítsd: [mi változzon]" (pl. "EP03 és EP05 cserélj", "EP07 legyen Depth")
Teljes újragenerálás → "újra"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Do NOT proceed to S2 until the concept and sequence are locked. This is the foundation document.

---

## STEP S2 — Agent 02: Per-Video Keyword Map (Playlist Keyword Mode)

Invoke `agent-02-seo-title` with the CONTEXT BLOCK and this brief:

> "Run in **Playlist Keyword Mode**. Input: the locked video sequence from S1. For ALL [N]
> videos at once, produce a broad keyword map: per video — `primary_keyword` (playlist-level
> anchor), `search_intent`, 2–3 `secondary_keywords`, `angle` (the specific take, one sentence),
> and `authority_hook` (a known name/tool, or none). Use vidIQ MCP if available.
> Overlap between videos is fine — do NOT dedupe or run a cannibalization check. Layer B refines
> each keyword per video at production time.
> Do NOT write titles here — titles are Layer B, per video."

No gate here — the keyword map is folded into the Gate 1 concept the operator already approved.
Update artifact S2 → `done`.

---

## STEP S3 — Emit the Interface Artifacts

Per `layer-interface.md` §1 and §2, write to `01_Strategy/`:

### 3a. The PlaylistStrategy doc
`create_file(title: "PlaylistStrategy_[SeriesName]", contentMimeType: "text/plain",
parentId: "[01_Strategy subfolderID]", textContent: "[PLAYLIST STRATEGY v1 block]")`.
Fill `optin_url:` from any existing `optin_registry.json` entry for this `playlist_id`, else
from a legacy `_OptinURL_[Series]` file, else `pending — set by the T2 funnel`.

### 3b. One Video Brief Card per planned video
For each EP in the sequence, `create_file(title: "Brief_[SeriesSlug]_EP[NN]_[topic-slug]",
contentMimeType: "text/plain", parentId: "[01_Strategy subfolderID]",
textContent: "[VIDEO BRIEF CARD v1 block]")` with `status: planned`.
Every field comes from S1 + S2 — no blanks except genuinely optional ones.

### 3c. `playlist_brief.json` — the T1 → T2 handoff (`layer-interface.md` §7)
Write it to `01_Strategy/` so the T2 funnel team can start independently. **Then stop — do NOT
invoke any T2 skill.** T2 (CMO orchestrator / email-list-building) reads this file on its own.
```
create_file(title: "playlist_brief", contentMimeType: "application/json",
parentId: "[01_Strategy subfolderID]", textContent:
{
  "playlist_id": "[series-slug]",
  "playlist_topic": "[plain-language topic]",
  "primary_keyword": "[umbrella_keyword]",
  "avatar_pain_point": "[2-3 sentences, from the Avatar Card + transformation_arc]",
  "script_summary": "[3 sentences max — the playlist's through-line, NOT any full script]",
  "video_count": [N]
})
```
Re-plan mode: overwrite it (new file, same title) only if `playlist_topic` / `primary_keyword` /
`video_count` changed; otherwise leave it — the funnel is already keyed to it.

### 3d. Re-plan mode only
Overwrite the existing PlaylistStrategy doc (new file, same title — newest wins). For Brief
Cards: regenerate only those with `status: planned`; skip every `in_production` / `published`
card. Bump `last_replanned`.

Update artifact S3 → `done`. Then the handback.

---

## HANDBACK

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ PLAYLIST STRATÉGIA — KÉSZ
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Csatorna: [CHANNEL_NAME]
Playlist: [series_name] — [N] videó megtervezve

DRIVE:
→ PlaylistStrategy_[SeriesName]: [webViewLink]
→ [N] Video Brief Card: Brief_[slug]_EP01 … EP[N]  (mind: status = planned)
→ playlist_brief.json: [webViewLink]  (a T2 funnel csapat ezt olvassa)

KÖVETKEZŐ LÉPÉS (két külön feladat, nem láncoljuk itt):
- FUNNEL (traffic célpont): mondd "start" vagy "lead magnet"
  → a T2 CMO orchestrator / email-list-building felveszi a playlist_brief.json-t,
    felépíti a lead magnetet + landing oldalt, és kiírja az optin_registry.json-t
- VIDEÓGYÁRTÁS: mondd "új videó"
  → a video-production-orchestrator felveszi az EP01 Brief Cardot; az opt-in URL-t
    az optin_registry.json-ból olvassa, ha a T2 már kész (ha nem, placeholderrel megy)
- Analytics után átrendezés: "playlist újratervezés"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## GLOBAL COMMANDS · RESUME · ERROR HANDLING
Per `conventions.md` §4–§6. Resume: on a partial run, report which of S1/S2/S3 exist on Drive
(PlaylistStrategy doc present? how many Brief Cards?) and offer to continue from the first gap.
