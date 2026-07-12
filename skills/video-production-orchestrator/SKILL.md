---
name: video-production-orchestrator
description: Video-level pipeline controller — runs for EVERY video, end-to-end from topic to published YouTube upload. Reads Drive state ONCE at startup, builds a CONTEXT BLOCK, and passes it to every agent (no repeated Drive reads). Automatically injects the permanent opt-in URL into all publishing artifacts. Triggers ONLY when the user says "új videó", "start new video", "video pipeline", "kezdjük a videót", "run the video pipeline", or "new video cycle". Core chain: 02 → 03 → [manual ElevenLabs audio gate, API fallback built in] → 04 thumbnail brief → 05 publish. Agent 01 Strategy only runs when no PlaylistStrategy doc exists. Shorts, Repurposing, Community = post-publish only, never part of core pipeline.
---

# Video Production Orchestrator
**Role:** Video-level pipeline controller. Runs for every single video.
**Key behaviours:** Reads Drive state ONCE (context block), reads the opt-in URL automatically
and injects it into Agent 05 publishing artifacts — zero manual coordination.
**Core chain:** 02 → 03 → [Manual Audio Gate] → 04 → 05 (5 gates)
**Agent 01 Strategy:** Conditional — only if no PlaylistStrategy doc exists for the active playlist.
**Audio (03b):** Manual gate — ElevenLabs is run by operator, not via API. Gate 4 = confirm audio is done.
**Thumbnail (04):** Brief + Nano Banana 2 prompt only. Canva MCP = optional, operator-triggered.
**Shorts / Repurposing / Community:** Post-publish menu ONLY. Never part of core pipeline.

---

## PIPELINE MAP

```
START
  │
  ▼
[00] Read active channel + scan Drive state + read opt-in URL
  │
  ▼
[01] Agent 01 — Video Strategy          ← CONDITIONAL: only if no PlaylistStrategy doc
  │  (skip if playlist plan already exists → go straight to Gate 1 topic confirm)
  ▼
GATE 1: Confirm topic / next video in sequence
  │
  ▼
[02] Agent 02 — SEO + Title             ← GATE 2: Confirm title
  │
  ▼
[03] Agent 03 — Script                  ← GATE 3: Approve script
  │
  ▼
[MANUAL] ElevenLabs audio by operator   ← GATE 4: "Audio kész" / "mehet"
  │  (operator pastes script into ElevenLabs, generates MP3, says done)
  ▼
[04] Agent 04 — Thumbnail brief + NB2   ← GATE 5: Approve thumbnail brief
  │  (Nano Banana 2 primary · Canva MCP only if operator requests)
  ▼
[05] Agent 05 Phase 1 — Publish         ← GATE 6: HeyGen render
  │
  ▼
[05b] Agent 05 Phase 2 — Chapters       ← trigger: "HeyGen jóváhagyva"
  │
  ▼
POST-PUBLISH MENU (all optional, never blocking)
  Shorts [S] · Repurposing [R] · Community [C] · Email/Lead Magnet [E] · Multilingual Captions [M]
```


---

## SHARED CONVENTIONS — READ FIRST
At startup, read `../../references/conventions.md` (plugin root `references/conventions.md`).
It defines: the CONTEXT BLOCK schema (§1), the shared status artifact HTML base (§2),
gate block format (§3), global commands (§4), resume logic (§5), and error handling (§6).
All of those apply to this orchestrator — they are not repeated below.

---

## STEP 0 — STARTUP: Read Channel + Drive State + Opt-In URL

### 0a. Verify active channel
1. `search_files(query: "title = '_Active_Channel'")` → first result → `read_file_content(fileId)`
2. Parse line 1: `[Channel Name] | [Channel Folder ID]`
3. If missing or empty → invoke `channel-manager` skill before proceeding.

### 0b. Read opt-in URL (CRITICAL — new in this orchestrator)
`search_files(query: "title contains '_OptinURL' and parentId = '[channelFolderID]'")` → read content.
Parse: `[SeriesName] | [OPTIN_URL] | confirmed: [DATE]`

If opt-in URL found → store as `ACTIVE_OPTIN_URL`. Pass to Agent 05 automatically at Step 6.
If NOT found → display warning:
```
⚠️  Nincs opt-in URL. A funnel még nincs felépítve.
   Javaslat: előbb futtasd a Strategic Launch Orchestratort.
   Folytatás opt-in URL nélkül? → "igen, folytatom" / "előbb funnel"
```
If operator says "igen, folytatom" → proceed with `ACTIVE_OPTIN_URL = null` (placeholder in descriptions).

### 0c. Scan current pipeline state
Check what's already done for the active channel this cycle:
- `01_Strategy` → any Video Brief Card from today or yesterday?
- `02_SEO_Titles` → any SEO doc with matching topic?
- `03_Scripts` → any approved script?
- `04_Thumbnail_Briefs` → any thumbnail brief from today?
- `05_Publish_Checklists` → any publish checklist?
- Local `Audio\` → any MP3 in `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`?

Use `search_files(query: "parentId = '[folderID]'")` for each, sort by `createdTime` descending.
Mark completed steps as DONE in artifact, offer skip with `"újra [N]"` to re-run.

### 0d. Build the CONTEXT BLOCK (one Drive scan — no repeated reads later)
Resolve ALL subfolder IDs in one call: `search_files(query: "parentId = '[channelFolderID]'")`.
Read the Foundation doc once, extract a 2-3 sentence avatar summary.
Assemble the CONTEXT BLOCK per conventions.md §1 (channel name + folder ID, all subfolder IDs,
opt-in URL, playlist info, lead magnet, avatar summary, confirmed title/topic as they become known).

**Pass the CONTEXT BLOCK at the top of EVERY agent invocation (Steps 1, 2, 3, 5, 6).**
Agents receiving it skip their own Pre-Flight Drive reads. Update the block's
CONFIRMED_TITLE / CONFIRMED_TOPIC fields as gates are passed.

### 0e. Create Pipeline Status Artifact
Create immediately via `mcp__cowork__create_artifact`. Update after every step.


## PIPELINE STATUS ARTIFACT (HTML)

Use the shared HTML base from `references/conventions.md` §2 (CSS + skeleton).
Header: `🎬 YT Pipeline`. Step rows for this orchestrator:

```html
<h2>🎬 YT Pipeline</h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --></div>
<div class="optin">🔗 Opt-In URL: <!-- OPTIN_URL --></div>
<div class="step <!-- S01 -->"><span class="icon">🧠</span>
<span class="label">01 — Strategy</span><span class="status"><!-- S01S --></span></div>
<div class="gate">▸ Gate 1: Confirm topic</div>
<div class="step <!-- S02 -->"><span class="icon">🔍</span>
<span class="label">02 — SEO + Title</span><span class="status"><!-- S02S --></span></div>
<div class="gate">▸ Gate 2: Confirm title</div>
<div class="step <!-- S03 -->"><span class="icon">📝</span>
<span class="label">03 — Script</span><span class="status"><!-- S03S --></span></div>
<div class="gate">▸ Gate 3: Approve script</div>
<div class="step <!-- S03b -->"><span class="icon">🎙️</span>
<span class="label">Audio — ElevenLabs (manuális)</span><span class="status"><!-- S03bS --></span></div>
<div class="gate">▸ Gate 4: Generálj ElevenLabs-ban → "audio kész"</div>
<div class="step <!-- S04 -->"><span class="icon">🖼️</span>
<span class="label">04 — Thumbnail Brief + Nano Banana 2</span><span class="status"><!-- S04S --></span></div>
<div class="gate">▸ Gate 5: Brief kész → generálj Nano Banana 2-vel (saját fotóval!)</div>
<div class="step <!-- S05 -->"><span class="icon">🚀</span>
<span class="label">05 — Publish + HeyGen Prompt</span><span class="status"><!-- S05S --></span></div>
<div class="gate">▸ Gate 6: HeyGen render → "HeyGen jóváhagyva"</div>
<div class="step <!-- S05b -->"><span class="icon">⏱️</span>
<span class="label">05b — Chapter Timestamps</span><span class="status"><!-- S05bS --></span></div>
```

Note the `<!-- OPTIN_URL -->` slot — populate from `ACTIVE_OPTIN_URL` found in Step 0b.


---

## STEP 1 — Agent 01: Video Strategy (CONDITIONAL)

**Check first:** Does a `PlaylistStrategy_[SeriesName]` doc already exist in `01_Strategy/`?
- `search_files(query: "title contains 'PlaylistStrategy' and parentId = '[01_Strategy subfolderID]'")`

**If PlaylistStrategy EXISTS:** Skip Agent 01. Read the next video in sequence from the strategy doc.
Display to operator: "📋 Playlist strategy found — next video: [topic]. Confirm → 'ok' or choose different topic."
Go directly to Gate 1 confirmation. Do NOT invoke Agent 01.

**If PlaylistStrategy does NOT exist:** Invoke `agent-01-channel-strategist` skill in full.
Agent 01 only runs when a new playlist needs to be planned — not per video.

After topic is ready, update artifact (01 → `waiting`):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 1 — Téma megerősítése
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Melyik témát választod? → sorszám
Saját téma → "téma: [leírás]"
Kihagyás (1. opció) → "skip"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Wait for operator response. Do NOT proceed to Agent 02 until topic is confirmed.

---

## STEP 2 — Agent 02: SEO + Title

Pass confirmed topic to Agent 02. Invoke `agent-02-seo-title` skill in full.
After Agent 02 writes the SEO doc to Drive, update artifact (02 → `waiting`):

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

## STEP 3 — Agent 03: Script

Pass confirmed title + keyword to Agent 03. Invoke `agent-03-script-writer` skill in full.
After script is written to Drive AND displayed in chat, update artifact (03 → `waiting`):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 3 — Script jóváhagyás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Módosítás → "módosítsd: [rész] → [változtatás]"
Teljes újragenerálás → "újra"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**This is the most critical gate.** Do NOT proceed until explicit script approval.

---

## STEP 4 — Manual Audio Gate (ElevenLabs by operator)

**Audio is generated manually by the operator in ElevenLabs.** (No separate audio agent exists — the API fallback below is built into this orchestrator.)

Display this gate to operator immediately after script approval:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 4 — ElevenLabs Audio (manuális)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
A script Drive-on van → másold be ElevenLabs-ba.
Generáld le az MP3-at a klónozott hanggal.
Mentsd ide: YouTube\Audio\

Ha kész → "audio kész" / "mehet" / "ok"
Ha ElevenLabs API-t akarsz helyette → "11labs api"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Update artifact (03b → `waiting`). Wait for operator confirmation.

**If operator says "11labs api" — inline API fallback (former Agent 03b, now built in):**
1. Read the approved script from `03_Scripts/` (or use CONTEXT BLOCK if already loaded).
2. Extract clean spoken text: strip `[SECTION]`/`[HOOK]`/`[B-ROLL:]`/`[RECORDING:]` markers,
   everything from `[THUMBNAIL CAPTURE NOTE]` onward, and gesture notes. Replace colons/em
   dashes/double hyphens with commas; "I am" → "I'm".
3. Generate via ElevenLabs API (`mcp__Desktop_Commander__start_process`, PowerShell):
   endpoint `https://api.elevenlabs.io/v1/text-to-speech/[VOICE_ID]/with-timestamps`,
   Voice ID `6cr2WtfrqWUvTgXGIdDB` (Zoltan BD 03 instant), model `eleven_turbo_v2_5`,
   API key from `C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env`.
   Save MP3 + `_align.json` to `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\`,
   naming: `[YYYY-MM-DD]_Audio_[title-slug]_BD03.mp3`.
4. Generate SRT: run `C:\Users\gdp\Documents\Claude\Projects\YouTube\Scripts\generate_srt.py`
   with the align file (offset 0; for >2500-char scripts split at sentence boundaries and
   chain offsets from each part's "Last seconds" value).
5. Present MP3 + SRT `computer://` links for approval, then continue to Gate 4 confirmation.

Do NOT proceed to Agent 04 until audio is confirmed.


---

## STEP 5 — Agent 04: Thumbnail Brief + Nano Banana 2

Invoke `agent-04-thumbnail` skill. It writes the brief to Drive and outputs the Nano Banana 2 prompt.
Agent 04 does NOT auto-call Canva MCP — brief + NB2 prompt is the primary deliverable.
Update artifact (04 → `waiting`):

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

Do NOT proceed to Agent 05 until operator confirms thumbnail is ready.
If operator says "canva": Agent 04 invokes Canva MCP (best-effort, failure = SKIPPED, not blocking).

---

## STEP 6 — Agent 05 Phase 1: Publish + HeyGen Prompt

Invoke `agent-05-launch-publish` skill Phase 1 in full.

**Opt-in URL injection (new):** Pass `ACTIVE_OPTIN_URL` to Agent 05 explicitly:
> "Az opt-in URL: [ACTIVE_OPTIN_URL] — ezt ágyazd be a YouTube description CTA blokkba
> és a Publishing Artifact minden releváns mezőjébe."

If `ACTIVE_OPTIN_URL` is null: Agent 05 inserts `[OPT-IN URL — FUNNEL STILL PENDING]`
as a placeholder in all description CTAs.

Agent 05 Phase 1 delivers:
- YouTube description (with live opt-in URL embedded in CTA block)
- Tags, draft chapters, pre-upload checklist
- HeyGen Production Prompt (saved to Drive)
- HTML Publishing Artifact (copy buttons)

After Publishing Artifact is created, update artifact (05 → `waiting`):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 6 — HeyGen render
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Publishing Artifact kész. HeyGen Prompt mentve Drive-ra.

Teendők:
1. Nyisd meg a HeyGen promptot Drive-ból
2. Illeszd be HeyGen Video Agent-be → renderelj
3. Nézd meg a videót
4. Ha rendben: "HeyGen jóváhagyva"
5. Ha script módosítás kell: "újra script [változtatás]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Wait for "HeyGen jóváhagyva" / "heygen ok" / "videó kész" / "approved" before Phase 2.

---

## STEP 7 — Agent 05 Phase 2: Chapter Timestamps

Trigger: "HeyGen jóváhagyva" (or equivalent).
Invoke Agent 05 Phase 2 in full. Finalizes chapter timestamps, updates Publishing Artifact.
Update artifact: 05b → `done`.

Then present the **Post-Publish Menu**:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ PIPELINE KÉSZ
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[S] Shorts    — 03c: script + HeyGen brief + YouTube upload
[R] Repurpose — 07: email snippet, LinkedIn, clip brief
[C] Community — 08: pinned comment, Community tab posts
[E] Email     — 09: conversion audit, CTA script check
[M] Captions  — 05b: 25 language SRT translation + YouTube upload (free, ~3 min)

→ Betű(k): pl. "S M" = Shorts + Multilingual Captions
→ "kész" = lezárás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```


---

## POST-PUBLISH AGENTS

**[S] Shorts — Agent 03c:** Invoke `agent-03c-shorts-producer`. Identifies 3-5 Short segments,
writes scripts + HeyGen briefs, saves to Drive, uploads finished Shorts to YouTube via Chrome.
Operator generates HeyGen Shorts manually, then says "Shorts kész, töltsd fel" to trigger upload.

**[R] Repurposing — Agent 07:** Invoke `agent-07-repurposing` in Repurposing mode. Outputs email
snippet brief + LinkedIn post brief to `07_Repurposing_Briefs/`. (Shorts belong exclusively to 03c.)

**[C] Community — Agent 07:** Invoke `agent-07-repurposing` in Community mode. Outputs pinned
comment template, Community tab post schedule, topic mining log — saved to `08_Community/`.

**[E] Conversion — Agent 09:** Invoke `agent-09-conversion-bridge` in audit mode.
Checks CTA alignment with current video. Runs conversion audit if lead magnet exists.

**[M] Multilingual Captions — Agent 05b:** Invoke `agent-05b-multilingual-captions` skill.
Translates English SRT to 25 languages via free Python script (deep-translator, no API key),
saves all SRT files to Drive Captions/ folder, uploads to YouTube as subtitle tracks.
SEO impact: video indexed in 25 languages. Cost: $0.00. Runtime: ~3 minutes.

---

## RESUME LOGIC · GLOBAL COMMANDS · ERROR HANDLING

Per `references/conventions.md` §4–§6. No orchestrator-specific deviations.

---

## OPERATOR EXPERIENCE SUMMARY

```
"új videó"
→ Artifact megjelenik · opt-in URL: ✓ (ha funnel kész)
→ [01 Strategy csak ha nincs PlaylistStrategy doc] → Gate 1: "2" (következő téma)
→ Agent 02 fut → Gate 2: "B" (cím)
→ Agent 03 fut → Gate 3: "jóváhagyom" (script)
→ [Manuális] ElevenLabs audio → Gate 4: "audio kész"
→ Agent 04 fut → Gate 5: Nano Banana 2 brief → thumbnail generálás → "thumbnail ok"
→ Agent 05 fut → Gate 6: HeyGen render → "HeyGen jóváhagyva"
→ Phase 2 kész → Post-publish menu → "S" / "R" / "C" / "E" / "M" → "kész"

Összes operátori input egy teljes videó ciklushoz: 7-8 rövid válasz.
Agent 01 = csak új playlistnél. 03b = manuális ElevenLabs. Thumbnail = Nano Banana 2.
Shorts/Repurposing/Community = post-publish, soha nem blokkoló.
```
