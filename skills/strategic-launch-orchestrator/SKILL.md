---
name: strategic-launch-orchestrator
description: Channel-level launch controller — runs ONCE per playlist before any video is published. Defines the full playlist concept, builds the lead magnet PDF, creates the landing page, sets up GHL automation, and stores the permanent opt-in URL so every video description has a live link from day one. Triggers when the user says "playlist launch", "channel launch", "new playlist", "playlist setup", "indítsuk el a playlistet", "launch strategy", "funnel setup", "build the funnel", "lead magnet előbb", "funnel először", "opt-in URL kell", "email rendszer setup", "playlist concept", "teljes launch", "indítsuk el a rendszert", or "before the first video".
---

# Strategic Launch Orchestrator
**Role:** Channel-level master controller. Runs ONCE per playlist, before video 1 is published.
**Chains:** Agent 01 (Playlist Mode) → Agent 09 (Conversion Bridge) — full funnel built in one session.
**Output:** Permanent opt-in URL stored in Drive → Video Production Orchestrator reads it automatically.

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
[S2] Agent 09 Phase 1 — Lead Magnet PDF   ← GATE 2: Approve content
  │
  ▼
[S3] Agent 09 Phase 2 — PDF Design (Canva) ← GATE 3: Approve design
  │
  ▼
[S4] Agent 09 Phase 3 — Landing Page HTML
  │
  ▼
[S5] Agent 09 Phase 4 — GHL Automation    ← GATE 4: Confirm GHL live + URL
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

```html
<!DOCTYPE html><html><head><meta charset="UTF-8"><style>
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;
background:#0f0f0f;color:#e0e0e0;padding:24px;max-width:540px;margin:0 auto}
h2{color:#fff;font-size:18px;margin-bottom:4px}
.ch{color:#888;font-size:13px;margin-bottom:20px}
.step{display:flex;align-items:center;gap:12px;padding:10px 14px;
border-radius:8px;margin-bottom:6px;background:#1a1a1a}
.step.done{background:#0d2b1a;border-left:3px solid #22c55e}
.step.active{background:#1a1a2e;border-left:3px solid #818cf8}
.step.waiting{background:#2b1a0d;border-left:3px solid #f59e0b}
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

## STEP S2 — Agent 09 Phase 1: Lead Magnet Content

Invoke `agent-09-conversion-bridge` skill, Phase 1 only (content writing — no design yet).

Pass as context:
- Full playlist strategy (Drive doc from S1)
- Avatar Card (from 00_Foundation)
- Series name and lead magnet transformation promise (from S1)

Agent 09 Phase 1 writes `LeadMagnet_[SeriesName]_RAW` to `09_Lead_Magnets/` in Drive.
Update artifact: S2 → `waiting`. Then present:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 2 — Lead Magnet tartalom jóváhagyás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead Magnet RAW mentve → [webViewLink]
Nézd át: tartalom, struktúra, fejezetcímek.
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Fejezet módosítás → "módosítsd [N]. fejezet: [változtatás]"
Teljes újraírás → "újra"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Do NOT proceed to S3 until content is approved.**
Design is built from this content — changes after design = wasted Canva credits.


---

## STEP S3 — Agent 09 Phase 2: PDF Design (Canva)

Invoke `agent-09-conversion-bridge` skill, Phase 2 only (Canva MCP design + export).

Agent 09 Phase 2 designs the PDF in Canva and exports it as
`LeadMagnet_[SeriesName]_designed.pdf` to `09_Lead_Magnets/` in Drive.

Update artifact: S3 → `waiting`. Then present:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ GATE 3 — PDF design jóváhagyás
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Canva design exportálva → [webViewLink]
Nézd át: brandszínek, tipográfia, layout.
Jóváhagyás → "jóváhagyom" / "ok" / "mehet"
Design módosítás → "módosítsd: [mi változzon]"
Manuális alternatíva → "manuálisan csinálom"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If "manuálisan csinálom": output full design brief (per-page colour/layout/typography spec)
for manual Canva execution. Continue to S4 after operator confirms PDF is ready.

---

## STEP S4 — Agent 09 Phase 3: Landing Page HTML

Runs automatically after Gate 3 approval — no additional gate.

Invoke `agent-09-conversion-bridge` skill, Phase 3 (HTML write + GHL publish).

Agent 09 Phase 3:
1. Writes production-ready `LandingPage_[SeriesName].html` to `09_Lead_Magnets/` in Drive
2. Publishes to GHL via Claude in Chrome (automated), OR outputs manual GHL steps if
   Chrome MCP is unavailable

If Claude in Chrome available → fully automated GHL page creation and publish.
If NOT available → output full HTML + step-by-step manual GHL UI instructions.

Update artifact: S4 → `done`. Proceed immediately to S5.

---

## STEP S5 — Agent 09 Phase 4: GHL Automation

Invoke `agent-09-conversion-bridge` skill, Phase 4 (GHL MCP automation setup).

Agent 09 Phase 4 executes via GHL MCP:
1. Upload designed PDF to GHL Media Library → retrieve hosted PDF URL
2. Create opt-in form (First Name, Email required; Phone optional)
3. Create delivery workflow: form submit → tag contact → immediate PDF delivery email
4. Run automated test (test@test.com)
5. Retrieve the permanent landing page URL

After GHL confirms live, update artifact: S5 → `waiting`. Then present:

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

## GLOBAL COMMANDS

| Command | Action |
|---------|--------|
| `"skip S[N]"` | Skip step (e.g. "skip S3" if PDF designed manually) |
| `"újra S[N]"` | Re-run specific step |
| `"hol tartunk"` | Show launch status artifact |
| `"állj meg"` | Pause, note current position |
| `"manuális S[N]"` | Output manual instructions instead of automating |

---

## ERROR HANDLING

- GHL MCP fails → output workflow config as structured brief + manual GHL UI steps.
- Canva MCP fails → output full per-page design brief for manual Canva execution.
- Chrome MCP unavailable → output full HTML file + manual GHL publish steps.
- Never block on tool failure. Surface errors explicitly — never skip silently.

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
