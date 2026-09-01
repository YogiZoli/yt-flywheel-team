---
name: video-production-orchestrator
description: LAYER B controller — runs the per-video pipeline end to end from an already-decided Video Brief Card to published YouTube upload. Consumes exactly ONE Brief Card (produced earlier by Layer A / playlist-strategy-orchestrator); does NO strategy, keyword, topic, angle or video-type decision at video time. Reads Drive state ONCE, builds a CONTEXT BLOCK (with BRIEF_ID), passes it to every agent. THREE video types via Production Core adapters — talking_head, heygen_avatar, screen_recording — shared PRE phase (load card, SEO title/description, master script) and shared POST phase (thumbnail, publish, chapters, post-publish menu). Triggers ONLY when the user says "új videó", "start new video", "video pipeline", "kezdjük a videót", "run the video pipeline", "new video cycle", or "ad-hoc videó" (playlist-less single video). Core chain: load Brief Card → 02 title/desc → 03 master script + type adapter → [PRODUCTION CORE] → 04 thumbnail → 05 publish. Shorts, Repurposing, Community = post-publish only.
---

# Video Production Orchestrator (v3 — Layer B, Brief-Card driven)
**Role:** Per-video pipeline controller (LAYER B). Runs for every single video, always inside
the frame a Video Brief Card already set.
**Key behaviours:** Resolves ONE Brief Card at Step 0, reads Drive state ONCE (context block
incl. `BRIEF_ID` + `VIDEO_TYPE`), reads the opt-in URL automatically and injects it into
Agent 05 publishing artifacts — zero manual coordination, zero strategy work.

**Layer boundary (see `references/layer-interface.md`):** this orchestrator NEVER invokes
Agent 01 and NEVER decides topic / keyword / angle / video_type / lead magnet. Those live on
the Brief Card. If no card exists, it stops and points the operator at
`playlist-strategy-orchestrator`, or runs the `"ad-hoc videó"` escape hatch.

**Architecture principle:**
```
COMMON PRE  →  PRODUCTION CORE (branch by VIDEO_TYPE)  →  COMMON POST
(type-independent)   (the ONLY place the pipeline forks)    (type-independent)
```
The **master script is the interface** between PRE and CORE. Agent 03 always produces one unified
master script; a type-specific **adapter** converts it into the final production format
(teleprompter / HeyGen brief / shot list). The fork happens at exactly one point — it never leaks
into PRE or POST.

**Video types:**

| video_type | Production format | Audio source | Production gate |
|---|---|---|---|
| talking_head | Teleprompter script | Live recording (camera mic) | Operator records + edits |
| heygen_avatar | HeyGen Production Prompt | ElevenLabs clone (manual/API) | HeyGen render approval |
| screen_recording | Shot list / screen script | ElevenLabs VO OR live narration | Recording + edit approval |

**Strategy:** never happens here. The Brief Card is the input. Missing card → stop or go ad-hoc.
**Thumbnail (04):** Brief + Nano Banana 2 prompt only. Canva MCP = optional, operator-triggered.
**Shorts / Repurposing / Community:** Post-publish menu ONLY. Never part of core pipeline.

---

## PIPELINE MAP

```
START
  ▼
[00] Resolve Brief Card + channel + scan Drive state + read opt-in URL (VIDEO_TYPE comes from the card)
  ▼
════════════════ COMMON PRE (type-independent) ════════════════
[01] Load Brief Card (read-only)        ← no Agent 01, no strategy. Missing card → stop / ad-hoc
  ▼
GATE 1: Confirm this is the card to run (topic + VIDEO_TYPE already on it)
  ▼
[02] Agent 02 — Execution Mode (title/desc/tags on the card's keyword)  ← GATE 2: Confirm title
  ▼
[03] Agent 03 — MASTER SCRIPT           ← GATE 3: Approve master script
  ▼
[03x] TYPE ADAPTER — converts master script to production format (automatic, no gate)
  ▼
═══════════ PRODUCTION CORE (branch by VIDEO_TYPE) ═══════════
  ├─ talking_head:     Teleprompter doc → GATE 4-TH: "felvétel kész"
  ├─ heygen_avatar:    [MANUAL] ElevenLabs audio → GATE 4-HG: "audio kész"
  │                    (HeyGen render gate comes later at Gate 6)
  └─ screen_recording: Shot list + VO script → GATE 4-SR: "felvétel kész"
  ▼
════════════════ COMMON POST (type-independent) ════════════════
[04] Agent 04 — Thumbnail brief + NB2   ← GATE 5: Approve thumbnail brief
  ▼
[05] Agent 05 Phase 1 — Publish         ← GATE 6: type-dependent final render/edit gate
  │    talking_head / screen_recording → "vágás kész"
  │    heygen_avatar → "HeyGen jóváhagyva"
  ▼
[05b] Agent 05 Phase 2 — Chapters       ← trigger: Gate 6 confirmation
  ▼
POST-PUBLISH MENU (all optional, never blocking)
  Shorts [S] · Repurposing [R] · Community [C] · Email/Lead Magnet [E] · Captions [M]
```

---

## SHARED CONVENTIONS — READ FIRST
At startup, read BOTH:
- `../../../references/conventions.md` — CONTEXT BLOCK schema (§1, incl. `BRIEF_ID` + VIDEO_TYPE),
  status artifact HTML base (§2), gate format (§3), global commands (§4), resume (§5), errors (§6).
- `../../../references/layer-interface.md` — the Brief Card v1 schema, Brief Card resolution
  order (§3), and the ad-hoc escape hatch (§4).
All of those apply to this orchestrator — they are not repeated below.

---

## STEP 0 — STARTUP: Brief Card + Channel + Drive State + Opt-In URL

### 0a. Resolve channel — CHANNEL PIN FIRST (conventions.md §7)
1. **Pin file:** look for `_Channel_Pin.txt` in the connected working folder. If found, parse
   `[Channel Name] | [LANG] | [Channel Folder ID]` → session is HARD-BOUND: use these values,
   write all public copy in [LANG], REFUSE "switch" commands, do NOT read `_Active_Channel`.
2. **Explicit channel in trigger** ("új videó @VoxFlow") → resolve that channel's folder ID from
   `YT_Flywheel/` and use it for this run.
3. **Legacy fallback:** `search_files(query: "title = '_Active_Channel'")` → read → parse line 1
   `[Channel Name] | [Channel Folder ID]`. Display a prominent channel badge (name + language)
   in the artifact AND in the Gate 1 message.
4. If nothing resolves → invoke `channel-manager` skill before proceeding.

### 0b. Read opt-in URL (CRITICAL)
`search_files(query: "title contains '_OptinURL' and parentId = '[channelFolderID]'")` → read content.
Parse: `[SeriesName] | [OPTIN_URL] | confirmed: [DATE]`

If found → store as `ACTIVE_OPTIN_URL`, pass to Agent 05 automatically at Step 6.
If NOT found → display warning:
```
⚠️  Nincs opt-in URL. A funnel még nincs felépítve.
   Javaslat: előbb futtasd a Strategic Launch Orchestratort.
   Folytatás opt-in URL nélkül? → "igen, folytatom" / "előbb funnel"
```
If operator says "igen, folytatom" → proceed with `ACTIVE_OPTIN_URL = null` (placeholder in descriptions).

### 0c. Resolve the Brief Card (the input — layer-interface.md §3)
First hit wins:
1. **Explicit in the trigger** — `"új videó EP04"` / `"új videó brief: youtube-ownership-ep04"`.
2. **Next `planned` card** — read `PlaylistStrategy_[Series]` in `01_Strategy/`, take the lowest-EP
   `Brief_*` card with `status: planned`. `search_files(parentId='[01_Strategy id]')`, read candidates.
3. **`"ad-hoc videó"` in the trigger** → §0c-adhoc below.
4. **No PlaylistStrategy doc, no card, not ad-hoc** → STOP:
   ```
   Nincs Video Brief Card ehhez a csatornához.
   → Playlist tervezés:  "playlist stratégia"
   → Egyszeri videó:     "ad-hoc videó"
   ```

Read the resolved card. `VIDEO_TYPE` = its `video_type`. Store `BRIEF_ID` = its `brief_id`.
`PLAYLIST` / arc / lead magnet / `next_episode_topic` all come from the card — do not re-derive.
Set the card `status: in_production` (new file, same title).

### 0c-adhoc. Ad-hoc single video (escape hatch — layer-interface.md §4)
Ask the operator exactly three things: topic, primary keyword (or `"kutasd ki"` → Agent 02 runs
its §C one-off research pass), video_type (T/H/S). Build a minimal Brief Card:
`playlist: AD-HOC`, `arc: n/a`, `cta_lead_magnet` = the channel's current `_OptinURL` lead magnet
if present else generic `email_optin`. Save as `Brief_ADHOC_[YYYY-MM-DD]_[slug]` with
`status: in_production`. Set `BRIEF_ID` accordingly. Continue the pipeline normally from Step 1.

### 0d. Scan current pipeline state
Check what's already done for the active channel this cycle:
- `01_Strategy` → any Video Brief Card from today or yesterday?
- `02_SEO_Titles` → any SEO doc with matching topic?
- `03_Scripts` → any approved master script? Any type-adapted output (`_TH` / `_HG` / `_SR`)?
- `04_Thumbnail_Briefs` → any thumbnail brief from today?
- `05_Publish_Checklists` → any publish checklist?
- Local `Audio\` → any MP3 in `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`? (HG / SR VO only)

Use `search_files(query: "parentId = '[folderID]'")` per folder, sort by `createdTime` descending.
Mark completed steps DONE in artifact, offer skip with `"újra [N]"`.

### 0e. Build the CONTEXT BLOCK v2 (one Drive scan — no repeated reads later)
Resolve ALL subfolder IDs in one call: `search_files(query: "parentId = '[channelFolderID]'")`.
Read the Foundation doc once, extract a 2-3 sentence avatar summary. Assemble the CONTEXT BLOCK
per conventions.md §1, **including** `BRIEF_ID` (from Step 0c), `VIDEO_TYPE`, `PLAYLIST`, and
`CONFIRMED_TOPIC` — all sourced from the Brief Card.

**Pass the CONTEXT BLOCK at the top of EVERY agent invocation (Steps 2, 3, 5, 6).**
Agents receiving it treat the `BRIEF_ID` card as read-only truth, skip their own Pre-Flight
Drive reads, and adapt output to `VIDEO_TYPE`. Update `CONFIRMED_TITLE` as Gate 2 is passed.

### 0f. Create Pipeline Status Artifact
Create immediately via `mcp__cowork__create_artifact`. Update after every step.

## PIPELINE STATUS ARTIFACT (HTML)
Use the shared HTML base from `references/conventions.md` §2 (CSS + skeleton).
Header: `🎬 YT Pipeline` + video type badge next to the date. The Step 4 row and Gate 6 label
render per VIDEO_TYPE (table below). Step rows:

```html
<h2>🎬 YT Pipeline</h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --> · 🎞 <!-- VIDEO_TYPE_LABEL --></div>
<div class="optin">🔗 Opt-In URL: <!-- OPTIN_URL --></div>
<div class="step <!-- S01 -->"><span class="icon">🗂️</span>
<span class="label">01 — Brief Card betöltve</span><span class="status"><!-- S01S --></span></div>
<div class="gate">▸ Gate 1: Confirm this Brief Card (topic + video type already on it)</div>
<div class="step <!-- S02 -->"><span class="icon">🔍</span>
<span class="label">02 — Title + Description (Execution Mode)</span><span class="status"><!-- S02S --></span></div>
<div class="gate">▸ Gate 2: Confirm title</div>
<div class="step <!-- S03 -->"><span class="icon">📝</span>
<span class="label">03 — Master Script</span><span class="status"><!-- S03S --></span></div>
<div class="gate">▸ Gate 3: Approve master script</div>
<div class="step <!-- S03x -->"><span class="icon"><!-- CORE_ICON --></span>
<span class="label"><!-- CORE_LABEL --></span><span class="status"><!-- S03xS --></span></div>
<div class="gate">▸ Gate 4: <!-- CORE_GATE_LABEL --></div>
<div class="step <!-- S04 -->"><span class="icon">🖼️</span>
<span class="label">04 — Thumbnail Brief + Nano Banana 2</span><span class="status"><!-- S04S --></span></div>
<div class="gate">▸ Gate 5: Brief kész → generálj Nano Banana 2-vel (saját fotóval!)</div>
<div class="step <!-- S05 -->"><span class="icon">🚀</span>
<span class="label">05 — Publish<!-- + HeyGen Prompt, only if heygen_avatar --></span><span class="status"><!-- S05S --></span></div>
<div class="gate">▸ Gate 6: <!-- FINAL_GATE_LABEL --></div>
<div class="step <!-- S05b -->"><span class="icon">⏱️</span>
<span class="label">05b — Chapter Timestamps</span><span class="status"><!-- S05bS --></span></div>
```

Type-dependent slots:

| VIDEO_TYPE | CORE_ICON / CORE_LABEL | CORE_GATE_LABEL | FINAL_GATE_LABEL |
|---|---|---|---|
| talking_head | 🎥 Teleprompter + felvétel | Felvétel kész → "felvétel kész" | Vágás kész → "vágás kész" |
| heygen_avatar | 🎙️ Audio — ElevenLabs (manuális) | Generálj ElevenLabs-ban → "audio kész" | HeyGen render → "HeyGen jóváhagyva" |
| screen_recording | 🖥️ Shot list + felvétel | Felvétel kész → "felvétel kész" | Vágás kész → "vágás kész" |

Note the `<!-- OPTIN_URL -->` slot — populate from `ACTIVE_OPTIN_URL` found in Step 0b.

---

# ═══════════ COMMON PRE PHASE (type-independent) ═══════════

## STEP 1 — Load the Brief Card (read-only, no strategy)
The card was already resolved and read at Step 0c. Nothing is decided here. Display its key
fields for confirmation:
```
📋 Brief Card: [brief_id]
   Playlist: [playlist] · [EP X of Y] · [arc]
   Téma: [topic]
   Keyword: [primary_keyword] ([search_intent]) · szög: [angle]
   Videó típus: [video_type]
   CTA lead magnet: [cta_lead_magnet]
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 1 — Ez a Brief Card fusson?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Másik kártya → "brief: [brief_id]" / "EP[NN]"
Ez stratégiai hiba (rossz téma/keyword/típus) → "újratervezés"
   → megnyitod: playlist-strategy-orchestrator, ez a pipeline áll
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
The card's `topic` / `primary_keyword` / `angle` / `video_type` are NOT editable here. A wrong
card is a Layer A problem: send the operator to `playlist-strategy-orchestrator` ("playlist
újratervezés"). The only exception is the ad-hoc card built at Step 0c-adhoc, which the operator
just defined. Do NOT proceed to Agent 02 until Gate 1 is confirmed.

---

## STEP 2 — Agent 02: Title + Description (EXECUTION MODE)
Invoke `agent-02-seo-title` with the CONTEXT BLOCK (`BRIEF_ID` set). It runs its §B Execution
Mode: reads the card's `primary_keyword` / `secondary_keywords` / `search_intent` / `angle` /
`authority_hook` verbatim and produces ONLY 3 title variants + description skeleton + tags +
hashtags + chapters placeholder. It does NOT research or change the keyword. (Ad-hoc card with
`primary_keyword: [PENDING]` → Agent 02 runs its §C one-off research pass instead.)
After the SEO doc is on Drive, update artifact (02 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 2 — Cím megerősítése
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Melyik cím? → A / B / C
Saját verzió → "cím: [szöveg]"
Kihagyás (A opció) → "skip"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## STEP 3 — Agent 03: MASTER SCRIPT
Pass confirmed title + keyword + VIDEO_TYPE to Agent 03. Invoke `agent-03-script-writer` in full.

**Master script = the interface.** Agent 03 writes ONE unified script (Ownership Framework),
with universal markers that every adapter understands:
- `[SECTION]`, `[HOOK]` — structure
- `[VISUAL: ...]` — type-neutral visual cue (adapter translates: B-roll / avatar gesture / screen action)
- `[SCREEN: ...]` — screen-action cue (only meaningful for screen_recording; others ignore it)
- Spoken text = plain paragraphs

Agent 03 may add a short type-flavored pass (e.g. tighter sentences for HeyGen TTS), but the
document structure is identical across types → PRE stays fully type-independent.

After script is written to Drive AND displayed in chat, update artifact (03 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 3 — Master script jóváhagyás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Módosítás → "módosítsd: [rész] → [változtatás]"
Teljes újragenerálás → "újra"
Típusváltás még lehetséges → "típus: T/H/S"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
**This is the most critical gate.** Do NOT proceed until explicit script approval.

---

## STEP 3x — TYPE ADAPTER (automatic, no gate)
Immediately after Gate 3 approval, convert the master script to the production format for
`VIDEO_TYPE`. Save the adapted doc to `03_Scripts/` with suffix `_TH` / `_HG` / `_SR`.

**Adapter: talking_head → Teleprompter doc**
- Strip all `[VISUAL:]` / `[SCREEN:]` markers into a separate "Rendezői jegyzet" appendix.
- Format spoken text in large-readable paragraphs, natural pause breaks (— / …).
- Add per-section retake markers (`● TAKE [n]`) for section-by-section recording.

**Adapter: heygen_avatar → HeyGen-ready clean text + audio prep**
- Extract clean spoken text: strip `[SECTION]`/`[HOOK]`/`[VISUAL:]`/`[SCREEN:]` markers,
  everything from `[THUMBNAIL CAPTURE NOTE]` onward, and gesture notes. Replace colons/em
  dashes/double hyphens with commas; "I am" → "I'm".
- Keep `[VISUAL:]` cues in a separate list → they feed the HeyGen Production Prompt at Step 6.

**Adapter: screen_recording → Shot list + VO script**
- Two-column production doc: LEFT = screen action (from `[SCREEN:]` + `[VISUAL:]` cues, numbered
  shots, app/URL/zoom notes), RIGHT = voiceover text aligned per shot.
- Read `sr_audio_mode` from the Brief Card (live | elevenlabs_vo). Only if the card says `n/a`,
  ask the operator once. Store as `SR_AUDIO_MODE`.

---

# ═══════ PRODUCTION CORE — branch by VIDEO_TYPE (Gate 4) ═══════

## STEP 4 — Production Gate (type-dependent)

### 4-TH · talking_head
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 4 — Felvétel (TalkingHead)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Teleprompter doc Drive-on (…_TH).
1. Nyisd meg teleprompter appban / második monitoron
2. Vedd fel szekciónként (● TAKE markerek)
3. Nyersanyag mentése a szokásos projektmappába
Ha kész → "felvétel kész"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
No audio sub-step — hang a felvételből jön. Update artifact (03x → `waiting`), wait for confirmation.

### 4-HG · heygen_avatar (manual ElevenLabs audio — API fallback built in)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 4 — ElevenLabs Audio (manuális)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
A clean script Drive-on (…_HG) → másold be ElevenLabs-ba.
Generáld le az MP3-at a klónozott hanggal.
Mentsd ide: YouTube\Audio\
Ha kész → "audio kész" / "mehet" / "ok"
Ha ElevenLabs API-t akarsz helyette → "11labs api"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
**If operator says "11labs api" — inline API fallback:**
1. Use the adapted clean text from Step 3x (…_HG doc) — no re-extraction needed.
2. Generate via ElevenLabs API (`mcp__Desktop_Commander__start_process`, PowerShell):
   endpoint `https://api.elevenlabs.io/v1/text-to-speech/[VOICE_ID]/with-timestamps`,
   Voice ID `6cr2WtfrqWUvTgXGIdDB` (Zoltan BD 03 instant), model `eleven_turbo_v2_5`,
   API key from `C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env`.
   Save MP3 + `_align.json` to `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`,
   naming: `[YYYY-MM-DD]_Audio_[title-slug]_BD03.mp3`.
3. Generate SRT: run `C:\Users\gdp\Documents\Claude\Projects\YouTube\Scripts\generate_srt.py`
   with the align file (offset 0; for >2500-char scripts split at sentence boundaries and
   chain offsets from each part's "Last seconds" value).
4. Present MP3 + SRT `computer://` links for approval, then continue to Gate 4 confirmation.

### 4-SR · screen_recording
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 4 — Képernyőfelvétel
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Shot list + VO script Drive-on (…_SR).
Audio mód: [SR_AUDIO_MODE]
• live: vedd fel a narrációt felvétel közben
• elevenlabs_vo: → "11labs api" vagy manuális generálás (mint HG ág), majd szinkron vágáskor
1. Menj végig a shot listán sorban
2. Nyersanyag mentése a szokásos projektmappába
Ha kész → "felvétel kész"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
If `SR_AUDIO_MODE = elevenlabs_vo`, the "11labs api" fallback above is available identically.

Do NOT proceed to Agent 04 until the Gate 4 confirmation for the active branch is received.

---

# ═══════════ COMMON POST PHASE (type-independent) ═══════════

## STEP 5 — Agent 04: Thumbnail Brief + Nano Banana 2
Invoke `agent-04-thumbnail`. It writes the brief to Drive and outputs the Nano Banana 2 prompt.
Agent 04 does NOT auto-call Canva MCP. Update artifact (04 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 5 — Thumbnail brief kész
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Brief + Nano Banana 2 prompt Drive-on.
→ Nyisd meg Nano Banana 2-t
→ Töltsd fel a saját profilképedet (kötelező!)
→ Illeszd be a promptot → generálj
Jóváhagyás → "thumbnail ok" / "ok" / "mehet"
Canva kell → "canva" (akkor Agent 04 meghívja)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Do NOT proceed to Agent 05 until thumbnail is confirmed. "canva" → Agent 04 invokes Canva MCP
(best-effort, failure = SKIPPED, not blocking).

---

## STEP 6 — Agent 05 Phase 1: Publish (+ HeyGen Prompt only for heygen_avatar)
Invoke `agent-05-launch-publish` Phase 1 in full. Pass VIDEO_TYPE in the CONTEXT BLOCK.

**Opt-in URL injection:** Pass `ACTIVE_OPTIN_URL` explicitly:
> "Az opt-in URL: [ACTIVE_OPTIN_URL] — ezt ágyazd be a YouTube description CTA blokkba
> és a Publishing Artifact minden releváns mezőjébe."

If `ACTIVE_OPTIN_URL` is null: Agent 05 inserts `[OPT-IN URL — FUNNEL STILL PENDING]` placeholder.

Agent 05 Phase 1 delivers (all types): YouTube description (opt-in URL in CTA block), tags,
draft chapters, pre-upload checklist, HTML Publishing Artifact (copy buttons).

**Type-specific extra:**
- `heygen_avatar` → HeyGen Production Prompt (from the …_HG doc + `[VISUAL:]` cue list), saved to Drive.
- `talking_head` / `screen_recording` → editing checklist instead (SRT sync, B-roll/zoom pass, intro trim).

After Publishing Artifact is created, update artifact (05 → `waiting`). Gate 6 by type:

**heygen_avatar:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 6 — HeyGen render
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Publishing Artifact kész. HeyGen Prompt mentve Drive-ra.
1. Nyisd meg a HeyGen promptot Drive-ból
2. Illeszd be HeyGen Video Agent-be → renderelj
3. Nézd meg a videót
4. Ha rendben: "HeyGen jóváhagyva"
5. Ha script módosítás kell: "újra script [változtatás]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Accept: "HeyGen jóváhagyva" / "heygen ok" / "videó kész" / "approved".

**talking_head / screen_recording:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 6 — Vágás / végleges videó
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Publishing Artifact kész. Editing checklist mentve Drive-ra.
1. Vágd meg a nyersanyagot a checklist szerint
2. Nézd át a végleges exportot
3. Ha rendben: "vágás kész" / "videó kész"
4. Ha újra kell venni egy szekciót: "retake: [szekció]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## STEP 7 — Agent 05 Phase 2: Chapter Timestamps
Trigger: Gate 6 confirmation (type-appropriate phrase). Invoke Agent 05 Phase 2 in full.
Finalizes chapter timestamps, updates Publishing Artifact. Update artifact: 05b → `done`.
**Flip the Brief Card `status: published`** (new file, same title) and, if it belongs to a
playlist, tick its row `brief: published` in `PlaylistStrategy_[Series]`.

Then present the **Post-Publish Menu**:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ PIPELINE KÉSZ
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[S] Shorts    — 03c: script + production brief + YouTube upload
[R] Repurpose — 07: email snippet, LinkedIn, clip brief
[C] Community — 07: pinned comment, Community tab posts
[E] Email     — 09: conversion audit, CTA script check
[M] Captions  — 05b: 25 language SRT translation + YouTube upload (free, ~3 min)
→ Betű(k): pl. "S M" = Shorts + Multilingual Captions
→ "kész" = lezárás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Note: for `talking_head` / `screen_recording` the [S] Shorts option produces timestamped clip-cut
briefs from the published video instead of HeyGen briefs — pass VIDEO_TYPE to Agent 03c.

---

## POST-PUBLISH AGENTS

**[S] Shorts — Agent 03c:** Invoke `agent-03c-shorts-producer` with VIDEO_TYPE.
heygen_avatar → HeyGen 9:16 briefs (as before). talking_head / screen_recording →
timestamped clip-cut briefs from the published long-form. Operator produces the Shorts,
then "Shorts kész, töltsd fel".

**[R] Repurposing — Agent 07:** Invoke `agent-07-repurposing` in Repurposing mode. Outputs email
snippet brief + LinkedIn post brief to `07_Repurposing_Briefs/`. (Shorts belong exclusively to 03c.)

**[C] Community — Agent 07:** Invoke `agent-07-repurposing` in Community mode. Outputs pinned
comment template, Community tab post schedule, topic mining log — saved to `08_Community/`.

**[E] Conversion — Agent 09:** Invoke `agent-09-conversion-bridge` in audit mode.
Checks CTA alignment with current video. Runs conversion audit if lead magnet exists.

**[M] Multilingual Captions — Agent 05b:** Invoke `agent-05b-multilingual-captions`.
Translates English SRT to 25 languages (free deep-translator script), saves to Drive Captions/,
uploads to YouTube as subtitle tracks. Cost: $0.00. Runtime: ~3 min.
(talking_head / screen_recording: source SRT comes from the edited video's captions —
generate with Whisper locally or download YouTube auto-captions if no ElevenLabs align file.)

---

## RESUME LOGIC · GLOBAL COMMANDS · ERROR HANDLING
Per `references/conventions.md` §4–§6. One v2 addition:
- Global command `"típus: T/H/S"` — allowed up to Gate 3 (inclusive). After Gate 3 approval,
  type change requires re-running the adapter: warn operator, re-run Step 3x, invalidate Gate 4+.

---

## BATCH MODE (safe because strategy is already done in Layer A)
Every `planned` Brief Card is a complete production order, so PRE needs no strategy input:
`"batch pre: 3"` → take the next 3 `planned` cards in the playlist sequence and run Steps 1–3
for each back-to-back (Gates 1–3 asked per video, grouped). Adapted production docs pile up in
`03_Scripts/`, each card flips to `status: in_production`, then the operator (or a VA) works
through the Production Cores asynchronously. Each video resumes independently at its own Gate 4.

---

## OPERATOR EXPERIENCE SUMMARY
```
"új videó"  (vagy "új videó EP04" / "ad-hoc videó")
→ Artifact megjelenik · opt-in URL: ✓ · típus badge (a Brief Cardból)
→ 01 Brief Card betöltve → Gate 1: "jóváhagyom" (a téma + típus már a kártyán van)
→ Agent 02 Execution Mode fut → Gate 2: "B" (cím)
→ Agent 03 fut → Gate 3: "jóváhagyom" (master script) → adapter automatikusan fut
→ PRODUCTION CORE:
   TH: teleprompter → "felvétel kész"
   HG: ElevenLabs → "audio kész"
   SR: shot list → "felvétel kész"
→ Agent 04 fut → Gate 5: Nano Banana 2 brief → "thumbnail ok"
→ Agent 05 fut → Gate 6: "HeyGen jóváhagyva" VAGY "vágás kész"
→ Phase 2 kész → Post-publish menu → "S" / "R" / "C" / "E" / "M" → "kész"

Összes operátori input egy teljes videó ciklushoz: 7-8 rövid válasz — típustól függetlenül.
```
