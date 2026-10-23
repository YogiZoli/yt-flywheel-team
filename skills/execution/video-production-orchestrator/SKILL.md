---
name: video-production-orchestrator
description: LAYER B controller — runs the per-video pipeline end to end from an already-decided Video Brief Card to published YouTube upload. Consumes exactly ONE Brief Card (produced earlier by Layer A / playlist-strategy-orchestrator); does NO strategy, keyword, topic, angle or video-type decision at video time. Reads Drive state ONCE, builds a CONTEXT BLOCK (with BRIEF_ID), passes it to every agent. FIVE video types via Production Core adapters, in priority order (most to least used) — avatar_local, avatar_native, screen_recording, talking_head, faceless_generated — with a shared PRE phase (load card, SEO title/description, master script) and shared POST phase (thumbnail, publish, chapters, post-publish menu). Triggers ONLY when the user says "new video", "start new video", "video pipeline", "run the video pipeline", "new video cycle", or "ad-hoc video" (playlist-less single video). Core chain: load Brief Card → 02 title/desc → 03 master script + type adapter → [PRODUCTION CORE] → 04 thumbnail → 05 publish. Shorts, Repurposing, Community = post-publish only.
---

# Video Production Orchestrator (v3 — Layer B, Brief-Card driven, 5 video types)
**Role:** Per-video pipeline controller (LAYER B). Runs for every single video, always inside
the frame a Video Brief Card already set.
**Key behaviours:** Resolves ONE Brief Card at Step 0, reads Drive state ONCE (context block
incl. `BRIEF_ID` + `VIDEO_TYPE`), reads the opt-in URL automatically and injects it into
Agent 05 publishing artifacts — zero manual coordination, zero strategy work.

**Layer boundary (see `references/layer-interface.md`):** this orchestrator NEVER invokes
Agent 01 and NEVER decides topic / keyword / angle / video_type / lead magnet. Those live on
the Brief Card. If no card exists, it stops and points the operator at
`playlist-strategy-orchestrator`, or runs the `"ad-hoc video"` escape hatch.

**Architecture principle:**
```
COMMON PRE  →  PRODUCTION CORE (branch by VIDEO_TYPE)  →  COMMON POST
(type-independent)   (the ONLY place the pipeline forks)    (type-independent)
```
The **master script is the interface** between PRE and CORE. Agent 03 always produces one unified
master script; a type-specific **adapter** converts it into the final production format
(teleprompter / HeyGen brief / shot list / generated-video script). The fork happens at exactly
one point — it never leaks into PRE or POST.

## VIDEO TYPES (enum — priority order, most to least used)

| # | video_type | Production format (adapter → suffix) | Audio source | Gate 4 (production) | Gate 6 (final) |
|---|---|---|---|---|---|
| 1 | `avatar_local` | HeyGen Production Prompt, local build + avatar clip layered (`_HG`) | VO (cloned or standard voice), manual/API | "audio done" | "HeyGen approved" |
| 2 | `avatar_native` | HeyGen Production Prompt, native Video Agent, motion graphics included (`_HG`) | VO (cloned or standard voice), manual/API | "audio done" | "HeyGen approved" |
| 3 | `screen_recording` | Shot list + VO script, Tella-style capture (`_SR`) | VO OR live narration | "recording done" | "edit done" |
| 4 | `talking_head` | Teleprompter doc, real selfie-style mobile recording (`_TH`) | Camera mic (live) | "recording done" | "edit done" |
| 5 | `faceless_generated` | Generated-video script, no avatar / no voice required (`_FL`) | optional VO or none | "generation done" | "edit done" |

- `avatar_local` is the primary, most-used mode. It took over the old `heygen_avatar` role and
  gate logic verbatim (VO → HeyGen render approval) — the rename `heygen_avatar → avatar_local`
  is the only change to that path.
- `avatar_native` follows the exact same gate pattern as `avatar_local`, but is **short-form /
  Shorts only** — reserve it for videos realistically under ~2.5 min (to stay under ~50 HeyGen
  credits). The orchestrator warns if a longer script is paired with `avatar_native`.
- `faceless_generated` follows the exact same gate pattern as `screen_recording` — a generated
  output review, no camera and no avatar approval step.
- `talking_head` can be a standalone video OR B-roll for another type.

**Avatar / voice are NOT hardcoded to one person.** For `avatar_local`, `avatar_native` and
`faceless_generated`, the pipeline is agnostic about (a) whether there is an avatar at all,
(b) whose it is — the operator's own, a standard HeyGen avatar, or a client's uploaded
avatar/voice clone, and (c) whether a cloned voice exists at all. These come from the channel
Foundation doc (`avatar_source`, `elevenlabs_voice_id` / `voice_source`) or a Brief Card
override, never from a fixed name or Voice ID in this file.

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
  ├─ avatar_local:       [MANUAL] VO audio → GATE 4-AL: "audio done"   (HeyGen render gate at Gate 6)
  ├─ avatar_native:      [MANUAL] VO audio → GATE 4-AN: "audio done"   (short-form only; HeyGen render gate at Gate 6)
  ├─ screen_recording:   Shot list + VO script → GATE 4-SR: "recording done"
  ├─ talking_head:       Teleprompter doc → GATE 4-TH: "recording done"
  └─ faceless_generated: Generated-video script → GATE 4-FL: "generation done"
  ▼
════════════════ COMMON POST (type-independent) ════════════════
[04] Agent 04 — Thumbnail brief + NB2   ← GATE 5: Approve thumbnail brief
  ▼
[05] Agent 05 Phase 1 — Publish         ← GATE 6: type-dependent final render/edit gate
  │    screen_recording / talking_head / faceless_generated → "edit done"
  │    avatar_local / avatar_native → "HeyGen approved"
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
2. **Explicit channel in trigger** ("new video @<channel>") → resolve that channel's folder ID from
   `YT_Flywheel/` and use it for this run.
3. **Legacy fallback:** `search_files(query: "title = '_Active_Channel'")` → read → parse line 1
   `[Channel Name] | [Channel Folder ID]`. Display a prominent channel badge (name + language)
   in the artifact AND in the Gate 1 message.
4. If nothing resolves → invoke `channel-manager` skill before proceeding.

### 0b. Read opt-in URL from the T2 contract (CRITICAL)
The opt-in URL is the T2 → T1 handoff (`references/layer-interface.md` §7). Resolve first hit:
1. **`optin_registry.json`** (T2 writes it): `search_files(query: "title contains 'optin_registry' and parentId = '[channelFolderID]'")`
   → read → find the entry whose `playlist_id` matches this run's playlist → take `opt_in_url`.
2. **Legacy `_OptinURL_[Series]`** text file (pre-contract): `search_files(query: "title contains '_OptinURL' and parentId = '[channelFolderID]'")`
   → parse `[SeriesName] | [OPTIN_URL] | confirmed: [DATE]`.

If found → store as `ACTIVE_OPTIN_URL`, pass to Agent 05 automatically at Step 6.
If NOT found → display warning:
```
WARNING: No opt-in URL (no optin_registry.json entry for this playlist).
   The funnel is built by the funnel skill ({integrations.funnel_skill}): say "start" or "lead magnet".
   Continue without an opt-in URL? → "yes, continue" / "funnel first"
```
If operator says "yes, continue" → proceed with `ACTIVE_OPTIN_URL = null` (placeholder in descriptions).

### 0c. Resolve the Brief Card (the input — layer-interface.md §3)
First hit wins:
1. **Explicit in the trigger** — `"new video EP04"` / `"new video brief: youtube-ownership-ep04"`.
2. **Next `planned` card** — read `PlaylistStrategy_[Series]` in `01_Strategy/`, take the lowest-EP
   `Brief_*` card with `status: planned`. `search_files(parentId='[01_Strategy id]')`, read candidates.
3. **`"ad-hoc video"` in the trigger** → §0c-adhoc below.
4. **No PlaylistStrategy doc, no card, not ad-hoc** → STOP:
   ```
   No Video Brief Card for this channel.
   → Plan a playlist:  "playlist strategy"
   → One-off video:    "ad-hoc video"
   ```

Read the resolved card. `VIDEO_TYPE` = its `video_type` (one of the 5 enum values). Store
`BRIEF_ID` = its `brief_id`. `PLAYLIST` / arc / lead magnet / `next_episode_topic` all come from
the card — do not re-derive. Set the card `status: in_production` (new file, same title).

**If `VIDEO_TYPE = avatar_native`**, sanity-check length: estimate script minutes (word count ÷
130). If likely over ~2.5 min, warn: `"avatar_native is for short content (under ~2.5 min).
This looks longer. Switch to avatar_local? → 'yes' / 'keep'"`.

### 0c-adhoc. Ad-hoc single video (escape hatch — layer-interface.md §4)
Ask the operator two things: topic and video_type (one of `avatar_local` / `avatar_native` /
`screen_recording` / `talking_head` / `faceless_generated` — offer the list, default
`avatar_local`). Leave `primary_keyword` blank — Agent 02 §B0 researches it at Step 2. Build a
minimal Brief Card: `playlist: AD-HOC`, `arc: n/a`, `cta_lead_magnet` = the lead magnet from any
`optin_registry.json` entry for the channel if present, else generic `email_optin`. Save as
`Brief_ADHOC_[YYYY-MM-DD]_[slug]` with `status: in_production`. Set `BRIEF_ID` accordingly.
Continue the pipeline normally from Step 1.

### 0d. Scan current pipeline state
Check what's already done for the active channel this cycle:
- `01_Strategy` → any Video Brief Card from today or yesterday?
- `02_SEO_Titles` → any SEO doc with matching topic?
- `03_Scripts` → any approved master script? Any type-adapted output (`_TH` / `_HG` / `_SR` / `_FL`)?
- `04_Thumbnail_Briefs` → any thumbnail brief from today?
- `05_Publish_Checklists` → any publish checklist?
- Local `Audio\` → any MP3 in `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`? (avatar_local / avatar_native / screen_recording VO only)

Use `search_files(query: "parentId = '[folderID]'")` per folder, sort by `createdTime` descending.
Mark completed steps DONE in artifact, offer skip with `"redo [N]"`.

### 0e. Build the CONTEXT BLOCK v2 (one Drive scan — no repeated reads later)
Resolve ALL subfolder IDs in one call: `search_files(query: "parentId = '[channelFolderID]'")`.
Read the Foundation doc once, extract a 2-3 sentence avatar summary **plus** the channel's
`avatar_source` and `voice_source` / `elevenlabs_voice_id` if present. Assemble the CONTEXT BLOCK
per conventions.md §1, **including** `BRIEF_ID` (from Step 0c), `VIDEO_TYPE` (one of the 5),
`PLAYLIST`, and `CONFIRMED_TOPIC` — all sourced from the Brief Card.

**Pass the CONTEXT BLOCK at the top of EVERY agent invocation (Steps 2, 3, 5, 6).**
Agents receiving it treat the `BRIEF_ID` card as read-only truth, skip their own Pre-Flight
Drive reads, and adapt output to `VIDEO_TYPE`. Update `CONFIRMED_TITLE` as Gate 2 is passed.

### 0f. Create Pipeline Status Artifact
Create immediately via `mcp__cowork__create_artifact`. Update after every step.

## PIPELINE STATUS ARTIFACT (HTML)
Use the shared HTML base from `references/conventions.md` §2 (CSS + skeleton).
Header: `YT Pipeline` + video type badge next to the date. The Step 4 row and Gate 6 label
render per VIDEO_TYPE (table below). Step rows:

```html
<h2>YT Pipeline</h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --> · <!-- VIDEO_TYPE_LABEL --></div>
<div class="optin">Opt-In URL: <!-- OPTIN_URL --></div>
<div class="step <!-- S01 -->"><span class="icon"></span>
<span class="label">01 — Brief Card loaded</span><span class="status"><!-- S01S --></span></div>
<div class="gate">▸ Gate 1: Confirm this Brief Card (topic + video type already on it)</div>
<div class="step <!-- S02 -->"><span class="icon"></span>
<span class="label">02 — Title + Description (Execution Mode)</span><span class="status"><!-- S02S --></span></div>
<div class="gate">▸ Gate 2: Confirm title</div>
<div class="step <!-- S03 -->"><span class="icon"></span>
<span class="label">03 — Master Script</span><span class="status"><!-- S03S --></span></div>
<div class="gate">▸ Gate 3: Approve master script</div>
<div class="step <!-- S03x -->"><span class="icon"><!-- CORE_ICON --></span>
<span class="label"><!-- CORE_LABEL --></span><span class="status"><!-- S03xS --></span></div>
<div class="gate">▸ Gate 4: <!-- CORE_GATE_LABEL --></div>
<div class="step <!-- S04 -->"><span class="icon"></span>
<span class="label">04 — Thumbnail Brief + Nano Banana 2</span><span class="status"><!-- S04S --></span></div>
<div class="gate">▸ Gate 5: Brief done → generate with Nano Banana 2</div>
<div class="step <!-- S05 -->"><span class="icon"></span>
<span class="label">05 — Publish<!-- + HeyGen Prompt, only if avatar_local / avatar_native --></span><span class="status"><!-- S05S --></span></div>
<div class="gate">▸ Gate 6: <!-- FINAL_GATE_LABEL --></div>
<div class="step <!-- S05b -->"><span class="icon"></span>
<span class="label">05b — Chapter Timestamps</span><span class="status"><!-- S05bS --></span></div>
```

Type-dependent slots:

| VIDEO_TYPE | CORE_ICON / CORE_LABEL | CORE_GATE_LABEL | FINAL_GATE_LABEL |
|---|---|---|---|
| avatar_local | VO audio (manual) | Generate VO → "audio done" | HeyGen render → "HeyGen approved" |
| avatar_native | VO audio (manual, short-form) | Generate VO → "audio done" | HeyGen native render → "HeyGen approved" |
| screen_recording | Shot list + recording | Recording done → "recording done" | Edit done → "edit done" |
| talking_head | Teleprompter + recording | Recording done → "recording done" | Edit done → "edit done" |
| faceless_generated | Generated script | Generation done → "generation done" | Edit done → "edit done" |

Note the `<!-- OPTIN_URL -->` slot — populate from `ACTIVE_OPTIN_URL` found in Step 0b.

---

# ═══════════ COMMON PRE PHASE (type-independent) ═══════════

## STEP 1 — Load the Brief Card (read-only, no strategy)
The card was already resolved and read at Step 0c. Nothing is decided here. Display its key
fields for confirmation:
```
Brief Card: [brief_id]
   Playlist: [playlist] · [EP X of Y] · [arc]
   Topic: [topic]
   Keyword (playlist anchor): [primary_keyword] ([search_intent]) · angle: [angle]
   (step 02 also runs a narrower, video-specific keyword research)
   Video type: [video_type]
   CTA lead magnet: [cta_lead_magnet]
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 1 — Ez a Brief Card fusson?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Approve → "approve" / "ok" / "go"
Another card → "brief: [brief_id]" / "EP[NN]"
This is a strategy error (wrong topic/angle/type/lead magnet) → "re-plan"
   → opens playlist-strategy-orchestrator, this pipeline stops
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
The card's `topic` / `primary_keyword` / `angle` / `video_type` are NOT editable here. A wrong
card is a Layer A problem: send the operator to `playlist-strategy-orchestrator` ("re-plan the
playlist"). The only exception is the ad-hoc card built at Step 0c-adhoc, which the operator
just defined. Do NOT proceed to Agent 02 until Gate 1 is confirmed.

---

## STEP 2 — Agent 02: Title + Description (EXECUTION MODE)
Invoke `agent-02-seo-title` with the CONTEXT BLOCK (`BRIEF_ID` set). It runs its §B Execution
Mode: takes the card's `primary_keyword` as a broad anchor, runs its own **tighter
video-specific keyword pass** on top (B0 — deliberate duplicate work: umbrella vs specific),
then produces 3 title variants + description skeleton + tags + hashtags + chapters placeholder.
If the card has no keyword it researches from scratch and continues — it never stops. It does
not touch `topic` / `angle` / `video_type` / `cta_lead_magnet`.
After the SEO doc is on Drive, update artifact (02 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 2 — Confirm the title
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Which title? → A / B / C
Own version → "title: [text]"
Skip (option A) → "skip"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## STEP 3 — Agent 03: MASTER SCRIPT
Pass confirmed title + keyword + VIDEO_TYPE to Agent 03. Invoke `agent-03-script-writer` in full.

**Master script = the interface.** Agent 03 writes ONE unified script (Ownership Framework),
with universal markers that every adapter understands:
- `[SECTION]`, `[HOOK]` — structure
- `[VISUAL: ...]` — type-neutral visual cue (adapter translates: B-roll / avatar gesture / screen action / generated shot)
- `[SCREEN: ...]` — screen-action cue (only meaningful for `screen_recording`; others ignore it)
- Spoken text = plain paragraphs

Agent 03 may add a short type-flavored pass (e.g. tighter sentences for TTS on the avatar and
faceless types), but the document structure is identical across types → PRE stays fully
type-independent.

After script is written to Drive AND displayed in chat, update artifact (03 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 3 — Approve the master script
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Approve → "approve" / "ok" / "go"
Change → "change: [part] → [change]"
Full regeneration → "redo"
Type change still possible → "type: [avatar_local | avatar_native | screen_recording | talking_head | faceless_generated]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
**This is the most critical gate.** Do NOT proceed until explicit script approval.

---

## STEP 3x — TYPE ADAPTER (automatic, no gate)
Immediately after Gate 3 approval, convert the master script to the production format for
`VIDEO_TYPE`. Save the adapted doc to `03_Scripts/` with suffix `_TH` / `_HG` / `_SR` / `_FL`.

**Adapter: avatar_local → HeyGen-ready clean text + audio prep** (suffix `_HG`)
- Extract clean spoken text: strip `[SECTION]`/`[HOOK]`/`[VISUAL:]`/`[SCREEN:]` markers,
  everything from `[THUMBNAIL CAPTURE NOTE]` onward, and gesture notes. Replace colons / em
  dashes / double hyphens with commas; "I am" → "I'm".
- Keep `[VISUAL:]` cues in a separate list → they feed the HeyGen Production Prompt at Step 6.

**Adapter: avatar_native → same as avatar_local** (suffix `_HG`), plus a header line
`SHORT-FORM ONLY — target < 2.5 min / < ~50 HeyGen credits`. Motion-graphics cues from
`[VISUAL:]` stay inline (the native Video Agent renders them).

**Adapter: screen_recording → Shot list + VO script** (suffix `_SR`)
- Two-column production doc: LEFT = screen action (from `[SCREEN:]` + `[VISUAL:]` cues, numbered
  shots, app/URL/zoom notes), RIGHT = voiceover text aligned per shot.
- Read `sr_audio_mode` from the Brief Card (live | elevenlabs_vo). Only if the card says `n/a`,
  ask the operator once. Store as `SR_AUDIO_MODE`.

**Adapter: talking_head → Teleprompter doc** (suffix `_TH`)
- Strip all `[VISUAL:]` / `[SCREEN:]` markers into a separate "Director notes" appendix.
- Format spoken text in large-readable paragraphs, natural pause breaks (— / …).
- Add per-section retake markers (`● TAKE [n]`) for section-by-section recording.

**Adapter: faceless_generated → Generated-video script** (suffix `_FL`)
- Two-column doc: LEFT = generated-shot description per beat (from `[VISUAL:]` cues — what the
  generated video should show), RIGHT = optional VO text per beat.
- Header line: `Avatar: none · Voice: [from Foundation voice_source, or "none / text-on-screen"]`.
- No `[SCREEN:]` handling — this type does not screen-record.

---

# ═══════ PRODUCTION CORE — branch by VIDEO_TYPE (Gate 4) ═══════

## STEP 4 — Production Gate (type-dependent)

### 4-AL · avatar_local (manual VO audio — API fallback built in)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 4 — VO Audio (manual)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
The clean script is on Drive (…_HG) → paste it into the voice generator (ElevenLabs or other).
Generate the MP3 with the channel voice (cloned or standard, see Foundation voice_source).
Mentsd ide: YouTube\Audio\
When done → "audio done" / "go" / "ok"
Ha ElevenLabs API-t akarsz helyette → "11labs api"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
**If operator says "11labs api" — inline API fallback:**
1. Use the adapted clean text from Step 3x (…_HG doc) — no re-extraction needed.
2. Generate via ElevenLabs API (`mcp__Desktop_Commander__start_process`, PowerShell):
   endpoint `https://api.elevenlabs.io/v1/text-to-speech/[VOICE_ID]/with-timestamps`,
   `[VOICE_ID]` = the channel's `elevenlabs_voice_id` from the Foundation doc (or the Brief
   Card's `voice_id` override). If the Foundation has no voice id, ask the operator for it once —
   do NOT assume a fixed voice. Model `eleven_turbo_v2_5`. API key from
   `C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env`.
   Save MP3 + `_align.json` to `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`,
   naming: `[YYYY-MM-DD]_Audio_[title-slug]_[voice-slug].mp3`.
3. Generate SRT: run `C:\Users\gdp\Documents\Claude\Projects\YouTube\Scripts\generate_srt.py`
   with the align file (offset 0; for >2500-char scripts split at sentence boundaries and
   chain offsets from each part's "Last seconds" value).
4. Present MP3 + SRT `computer://` links for approval, then continue to Gate 4 confirmation.

### 4-AN · avatar_native (short-form / Shorts only — identical gate to 4-AL)
Same as 4-AL. Extra reminder in the gate text: `WARNING: avatar_native: tartsd < 2.5 perc alatt
(~50 HeyGen credit). If longer → switch to avatar_local.` VO audio generated the same
way (Foundation voice, or "11labs api").

### 4-SR · screen_recording
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 4 — Screen recording
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Shot list + VO script Drive-on (…_SR).
Audio mode: [SR_AUDIO_MODE]
• live: record the narration while recording
• elevenlabs_vo: → "11labs api" or manual generation (like the avatar_local branch), then sync in the edit
1. Go through the shot list in order
2. Save the raw footage to the usual project folder
When done → "recording done"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
If `SR_AUDIO_MODE = elevenlabs_vo`, the "11labs api" fallback from 4-AL is available identically.

### 4-TH · talking_head
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 4 — Recording (TalkingHead)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Teleprompter doc Drive-on (…_TH).
1. Open it in a teleprompter app / on a second monitor
2. Record section by section (● TAKE markers), mobile selfie style
3. Save the raw footage to the usual project folder
When done → "recording done"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
No audio sub-step. The audio comes from the recording. Update artifact (03x → `waiting`), wait for confirmation.

### 4-FL · faceless_generated (generated output review — no camera, no avatar)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 4 — Generated video
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Generated-video script Drive-on (…_FL).
1. Put the shot descriptions into the video generator (the channel's chosen tool)
2. If there is VO: generate it per Foundation voice_source (or "11labs api"), or keep text-on-screen mode
3. Save the raw footage / generated clips to the usual project folder
When done → "generation done" / "recording done"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Do NOT proceed to Agent 04 until the Gate 4 confirmation for the active branch is received.

---

# ═══════════ COMMON POST PHASE (type-independent) ═══════════

## STEP 5 — Agent 04: Thumbnail Brief + Nano Banana 2
Invoke `agent-04-thumbnail`. It writes the brief to Drive and outputs the Nano Banana 2 prompt.
Agent 04 does NOT auto-call Canva MCP. Update artifact (04 → `waiting`):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 5 — Thumbnail brief done
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Brief + Nano Banana 2 prompt Drive-on.
→ Nyisd meg Nano Banana 2-t
→ If the channel has a personal face brand: upload the channel face photo ({brand.face_photo})
   (avatar_local / avatar_native / talking_head). faceless_generated → nincs arc.
→ Paste the prompt → generate
Approve → "thumbnail ok" / "ok" / "go"
Need Canva → "canva" (then Agent 04 calls it)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Do NOT proceed to Agent 05 until thumbnail is confirmed. "canva" → Agent 04 invokes Canva MCP
(best-effort, failure = SKIPPED, not blocking).

---

## STEP 6 — Agent 05 Phase 1: Publish (+ HeyGen Prompt only for avatar_local / avatar_native)
Invoke `agent-05-launch-publish` Phase 1 in full. Pass VIDEO_TYPE in the CONTEXT BLOCK.

**Opt-in URL injection:** Pass `ACTIVE_OPTIN_URL` explicitly:
> "The opt-in URL: [ACTIVE_OPTIN_URL]. Put it in the YouTube description CTA block
> and in every relevant field of the Publishing Artifact."

If `ACTIVE_OPTIN_URL` is null: Agent 05 inserts `[OPT-IN URL — FUNNEL STILL PENDING]` placeholder.

Agent 05 Phase 1 delivers (all types): YouTube description (opt-in URL in CTA block), tags,
draft chapters, pre-upload checklist, HTML Publishing Artifact (copy buttons).

**Type-specific extra:**
- `avatar_local` / `avatar_native` → HeyGen Production Prompt (from the …_HG doc + `[VISUAL:]`
  cue list), saved to Drive. The prompt names the avatar per the channel's `avatar_source`
  (own / standard HeyGen / client upload), not a fixed identity.
- `screen_recording` / `talking_head` / `faceless_generated` → editing checklist instead
  (SRT sync, B-roll/zoom pass, intro trim; for faceless_generated: shot-order + VO sync).

After Publishing Artifact is created, update artifact (05 → `waiting`). Gate 6 by type:

**avatar_local / avatar_native:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 6 — HeyGen render
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Publishing Artifact done. HeyGen Prompt saved to Drive.
1. Open the HeyGen prompt from Drive
2. Illeszd be HeyGen Video Agent-be → renderelj
   (avatar_native: native, short; avatar_local: local build + avatar clip layered on top)
3. Watch the video
4. If it is fine: "HeyGen approved"
5. If the script needs a change: "redo script [change]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Accept: "HeyGen approved" / "heygen ok" / "video done" / "approved".

**screen_recording / talking_head / faceless_generated:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GATE 6 — Edit / final video
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Publishing Artifact done. Editing checklist saved to Drive.
1. Edit the raw footage per the checklist
2. Review the final export
3. If it is fine: "edit done" / "video done"
4. If a section needs a retake: "retake: [section]"
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
PIPELINE DONE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[S] Shorts    — 03c: script + production brief + YouTube upload
[R] Repurpose — 07: email snippet, LinkedIn, clip brief
[C] Community — 07: pinned comment, Community tab posts
[E] Email     — 09: conversion audit, CTA script check
[M] Captions  — 05b: 25 language SRT translation + YouTube upload (free, ~3 min)
→ Letter(s): e.g. "S M" = Shorts + Multilingual Captions
→ "done" = close
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Note: for `screen_recording` / `talking_head` / `faceless_generated` the [S] Shorts option
produces timestamped clip-cut briefs from the published video instead of HeyGen briefs — pass
VIDEO_TYPE to Agent 03c.

---

## POST-PUBLISH AGENTS

**[S] Shorts — Agent 03c:** Invoke `agent-03c-shorts-producer` with VIDEO_TYPE.
`avatar_local` / `avatar_native` → HeyGen 9:16 briefs. `screen_recording` / `talking_head` /
`faceless_generated` → timestamped clip-cut briefs from the published long-form. Operator
produces the Shorts, then "Shorts done, upload".

**[R] Repurposing — Agent 07:** Invoke `agent-07-repurposing` in Repurposing mode. Outputs email
snippet brief + LinkedIn post brief to `07_Repurposing_Briefs/`. (Shorts belong exclusively to 03c.)

**[C] Community — Agent 07:** Invoke `agent-07-repurposing` in Community mode. Outputs pinned
comment template, Community tab post schedule, topic mining log — saved to `08_Community/`.

**[E] Conversion — Agent 09:** Invoke `agent-09-conversion-bridge` in audit mode.
Checks CTA alignment with current video. Runs conversion audit if lead magnet exists.

**[M] Multilingual Captions — Agent 05b:** Invoke `agent-05b-multilingual-captions`.
Translates English SRT to 25 languages (free deep-translator script), saves to Drive Captions/,
uploads to YouTube as subtitle tracks. Cost: $0.00. Runtime: ~3 min.
(`screen_recording` / `talking_head` / `faceless_generated`: source SRT comes from the edited
video's captions — generate with Whisper locally or download YouTube auto-captions if there is
no VO align file.)

---

## RESUME LOGIC · GLOBAL COMMANDS · ERROR HANDLING
Per `references/conventions.md` §4–§6. One addition:
- Global command `"type: [enum value]"` — allowed up to Gate 3 (inclusive). After Gate 3
  approval, a type change requires re-running the adapter: warn operator, re-run Step 3x,
  invalidate Gate 4+. Switching TO `avatar_native` also re-checks the length guard from Step 0c.

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
"new video"  (or "new video EP04" / "ad-hoc video")
→ Artifact megjelenik · opt-in URL: ok · type badge (from the Brief Card)
→ 01 Brief Card loaded → Gate 1: "approve" (topic + type are already on the card)
→ Agent 02 Execution Mode runs → Gate 2: "B" (title)
→ Agent 03 runs → Gate 3: "approve" (master script) → adapter runs automatically
→ PRODUCTION CORE:
   avatar_local / avatar_native: VO audio → "audio done"
   screen_recording:             shot list → "recording done"
   talking_head:                 teleprompter → "recording done"
   faceless_generated:           generated script → "generation done"
→ Agent 04 runs → Gate 5: Nano Banana 2 brief → "thumbnail ok"
→ Agent 05 runs → Gate 6: "HeyGen approved" OR "edit done"
→ Phase 2 done → Post-publish menu → "S" / "R" / "C" / "E" / "M" → "done"

Total operator input for one full video cycle: 7 to 8 short replies, for every video type.
```
