---
name: strategic-launch-orchestrator
description: Channel-level launch controller — runs ONCE per playlist before any video is published. Defines the full playlist concept, then delegates funnel building (lead magnet, landing page, GHL automation) to the email-list-building plugin's list-building-orchestrator, and stores the permanent opt-in URL so every video description has a live link from day one. Triggers ONLY when the user says "playlist launch", "channel launch", "new playlist", "playlist setup", "indítsuk el a playlistet", "playlist concept", "teljes launch", or "before the first video". (Funnel/lead-magnet-only requests belong to the email-list-building plugin, not here.)
---

# Strategic Launch Orchestrator
**Role:** Channel-level master controller. Runs ONCE per playlist, before video 1 is published.
**Chains:** Agent 01 (Playlist Mode) → `email-list-building:list-building-orchestrator` (funnel build)
→ Agent 09 (YouTube-side bridge: description CTAs + opt-in URL storage).
**Output:** Permanent opt-in URL stored in Drive → Video Production Orchestrator reads it automatically.

## SHARED CONVENTIONS — READ FIRST
At startup, read `../../references/conventions.md` (plugin root `references/conventions.md`).
It defines: the CONTEXT BLOCK schema (§1), the shared status artifact HTML base (§2),
gate block format (§3), global commands (§4), resume logic (§5), and error handling (§6).
Build the CONTEXT BLOCK at Step 0 and pass it to every skill invoked.

---

## WHY THIS EXISTS

YouTube videos die without a destination. This orchestrator ensures that when video 1 goes live:
- A full playlist concept exists (all 8–12 video topics defined)
- A lead magnet PDF is built (content written, designed, hosted)
- A landing page is live (HTML built, GHL published)
- A GHL delivery automation is running (opt-in → email → PDF → tag)
- A permanent opt-in URL is stored in Drive (never changes — all descriptions point to it)

The Video Production Orchestrator reads this URL automatically. Zero manual coordination needed.


---

## LAUNCH MAP

```
START
  │
  ▼
[00] Read active channel + Foundation doc
  │
  ▼
[S1] Agent 01 — Playlist Strategy Mode    ← GATE 1: Confirm playlist concept
  │
  ▼
[S2–S5] Funnel Build — DELEGATED to email-list-building plugin
  │      (list-building-orchestrator: lead magnet → design → landing page → GHL)
  │      ← its own gates run inside the T2 team; GATE 4 here: Confirm GHL live + URL
  │
  ▼
[S6] Store opt-in URL → Drive (_OptinURL file)
  │
  ▼
LAUNCH READY — Video Production Orchestrator can now start
```

---

## STEP 0 — STARTUP: Read Channel + Foundation

### 0a. Verify active channel
1. `search_files(query: "title = '_Active_Channel'")` → first result → `read_file_content(fileId)`
2. Parse line 1: `[Channel Name] | [Channel Folder ID]`
3. If missing or empty → invoke `channel-manager` skill before proceeding.


### 0b. Read Foundation
`search_files(query: "title = '00_Foundation' and parentId = '[channelFolderID]'")` → subfolder ID →
`search_files(query: "parentId = '[subfolderID]'")` → `read_file_content` on most recent doc.
Extract: Avatar Card (viewer identity, transformation), Hero Story, channel niche.

### 0c. Check for existing playlist strategy
`search_files(query: "title contains 'PlaylistStrategy' and parentId = '[channelFolderID]'")` → if found, read it.
If a strategy already exists: present it and ask:
"Folytatjuk ezt a playlistet, vagy új playlistet indítunk?"
- Continue → skip S1, go directly to S2.
- New → run S1 fresh.

### 0d. Check for existing opt-in URL
`search_files(query: "title contains '_OptinURL' and parentId = '[channelFolderID]'")` → if found, read content.
If a live opt-in URL exists: present it and ask:
"Ez az URL él: [URL]. Új funnelt építünk vagy ez megmarad?"
- Same URL → skip S2–S5, go directly to LAUNCH READY.
- New funnel → run full flow.

### 0e. Create Launch Status Artifact
Create via `mcp__cowork__create_artifact` with the HTML template below.
Update it after every completed step using `mcp__cowork__update_artifact`.


## LAUNCH STATUS ARTIFACT (HTML)

Use the shared HTML base from `references/conventions.md` §2 (CSS + skeleton).
Header: `🚀 Strategic Launch`. Step rows for this orchestrator:

```html
<h2>🚀 Strategic Launch</h2>
<div class="ch"><!-- CHANNEL_NAME --> · <!-- DATE --></div>
<div class="step <!-- S1 -->"><span class="icon">🧠</span>
<span class="label">S1 — Playlist Strategy</span>
<span class="status"><!-- S1S --></span></div>
<div class="gate">▸ Gate 1: Confirm playlist concept</div>
<div class="step <!-- S2 -->"><span class="icon">📄</span>
<span class="label">S2 — Lead Magnet Content</span>
<span class="status"><!-- S2S --></span></div>
<div class="gate">▸ Gate 2: Approve written content</div>
<div class="step <!-- S3 -->"><span class="icon">🎨</span>
<span class="label">S3 — PDF Design (Canva)</span>
<span class="status"><!-- S3S --></span></div>
<div class="gate">▸ Gate 3: Approve designed PDF</div>
<div class="step <!-- S4 -->"><span class="icon">🌐</span>
<span class="label">S4 — Landing Page HTML</span>
<span class="status"><!-- S4S --></span></div>
<div class="step <!-- S5 -->"><span class="icon">⚙️</span>
<span class="label">S5 — GHL Automation</span>
<span class="status"><!-- S5S --></span></div>
<div class="gate">▸ Gate 4: Confirm GHL live + opt-in URL</div>
<div class="step <!-- S6 -->"><span class="icon">💾</span>
<span class="label">S6 — Store Opt-In URL</span>
<span class="status"><!-- S6S --></span></div>
<div class="urlbox <!-- URL_VIS -->">
<div class="urllabel">✅ PERMANENT OPT-IN URL</div>
<div class="urlval"><!-- OPTIN_URL --></div></div>
</body></html>
```

Step classes: `done` / `active` / `waiting` / `pending`
Status text: `"✓ kész"` / `"→ fut..."` / `"⏸ jóváhagyásra vár"` / `"—"`


---

## STEP S1 — Agent 01: Playlist Strategy Mode

**NOT the standard "next video" mode. This is playlist-level thinking.**

Invoke `agent-01-channel-strategist` skill with the following modified brief:

> "Run in Playlist Strategy Mode. Do NOT pick one video. Instead, design the full playlist:
> - Playlist name and topic cluster
> - Umbrella keyword for the playlist
> - 8–12 video topics in order (Evergreen-first sequence)
> - Video type for each (Evergreen / Viral / Depth — target ratio: 65% / 25% / 10%)
> - `default_video_type:` for the playlist (talking_head / heygen_avatar / screen_recording)
>   plus a per-video `video_type:` where a video deviates from the default —
>   the Video Production Orchestrator reads these at Step 0c (no need to ask per video)
> - Lead magnet concept: the ONE transformation the companion guide delivers
> - Recommended CTA language (consistent across all videos in the series)"

Save Playlist Strategy to Drive: `PlaylistStrategy_[SeriesName]` in `01_Strategy/` subfolder.
Update artifact: S1 → `waiting`. Then present:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 1 — Playlist koncepció jóváhagyás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Playlist stratégia mentve → [webViewLink]
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Módosítás → "módosítsd: [mi változzon]"
Teljes újragenerálás → "újra"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Do NOT proceed to S2 until the playlist concept is locked.**
This is the foundation document — all downstream content derives from it.

---

## STEP S2–S5 — Funnel Build (DELEGATED — single source of truth)

**Do NOT build the funnel here.** Invoke the `email-list-building:list-building-orchestrator`
skill (T2 team) — it owns lead magnet content, PDF design, landing page, and GHL automation,
with its own internal approval gates.

Pass as context (CONTEXT BLOCK + these fields):
- Full playlist strategy (Drive doc from S1) + series name + lead magnet transformation promise
- Avatar Card summary (from 00_Foundation)
- Target Drive folder for outputs: this channel's `09_Lead_Magnets/` subfolder ID
- Required return values: permanent opt-in URL + hosted PDF URL + tag names

Requirements the T2 team must honour (state them explicitly in the handoff):
- One playlist = one lead magnet = one permanent opt-in URL (never changes)
- Contacts tagged `lead_magnet_[series_slug]` + `source_youtube`
- Automated test before confirming live

**If the email-list-building plugin is NOT installed:** tell the operator, and output the full
funnel spec (lead magnet outline, landing page copy, GHL workflow config) as a manual brief
instead — never silently skip.

While T2 runs, keep this launch artifact updated: S2–S5 → `active`, then `done` per T2 progress.


---

## GATE 4 — GHL Live Confirmation (after the T2 team returns)

When the list-building-orchestrator reports the funnel live (opt-in URL + hosted PDF URL + tags),
update artifact: S5 → `waiting`. Then present:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 4 — GHL live megerősítés
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GHL automation tesztelve (test@test.com ✓)
Landing page URL: [URL]

Manuálisan teszteld:
1. Nyisd meg a fenti URL-t
2. Töltsd ki a saját emaileddel
3. Ellenőrizd a PDF delivery emailt
4. Kattints a letöltési linkre

Minden OK → "él" / "live" / "ok"
Probléma → "probléma: [mi nem működik]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```


---

## STEP S6 — Store Opt-In URL to Drive

After Gate 4 confirmation, store the permanent opt-in URL so all future agents read it automatically.

```
create_file(
  title: "_OptinURL_[SeriesName]",
  textContent: "[SeriesName] | [OPTIN_URL] | confirmed: [DATE]",
  contentMimeType: "text/plain",
  parentId: "[09_Lead_Magnets subfolder ID]"
)
```

Update artifact: S6 → `done`, show the URL box (add `visible` class to `.urlbox`).

Then invoke `agent-09-conversion-bridge` (CTA mode) with the CONTEXT BLOCK + confirmed opt-in URL
to generate `CTA_Descriptions_[SeriesName]` for every video in the playlist.

---

## LAUNCH READY OUTPUT

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ STRATEGIC LAUNCH — KÉSZ
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Channel: [channel name]
Playlist: [series name] ([N] videó tervezve)

DRIVE OUTPUTS:
→ Playlist Strategy:      [webViewLink]
→ Lead Magnet RAW:        [webViewLink]
→ Lead Magnet Designed:   [webViewLink]
→ Landing Page HTML:      [webViewLink]
→ CTA Descriptions:       [webViewLink]

GHL LIVE:
→ Opt-In URL: [OPTIN_URL]  ← ez megy MINDEN videó leírásába
→ PDF delivery automation: ACTIVE
→ Contacts auto-tagged: lead_magnet_[series_slug] + source_youtube
→ Flow tesztelve és megerősítve ✓

KÖVETKEZŐ LÉPÉS:
→ Mondd: "új videó" → Video Production Orchestrator automatikusan
  beolvassa az opt-in URL-t és beépíti minden publishing artifactba.
→ Az első videót most már ki lehet adni — a funnel él.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## GLOBAL COMMANDS · ERROR HANDLING

Per `references/conventions.md` §4 and §6. Additional rule: if the email-list-building plugin
is missing, degrade to manual funnel brief (see Step S2–S5) — never block the playlist launch.

---

## RESUME LOGIC

If triggered and Drive scan shows partial completion, present:

```
Részleges launch található:
→ S1 Playlist Strategy: [kész / hiányzik]
→ S2 Lead Magnet RAW: [kész / hiányzik]
→ S3 PDF Design: [kész / hiányzik]
→ S4 Landing Page: [kész / hiányzik]
→ S5 GHL: [él / nincs]
→ S6 Opt-In URL: [URL / nincs]

Folytatás a következő hiányzó lépéstől? → "igen"
Elölről → "újrakezdem"
Adott lépés → "újra S[N]"
```
