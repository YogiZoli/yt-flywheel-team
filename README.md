# YouTube Evergreen Flywheel — AI Agent Team (Cowork Plugin)

A complete AI Agent Team that runs your full YouTube publishing workflow across **multiple channels**, with all outputs saved directly to **Google Drive**.

Built for Business Owners (not YouTubers). The email list is the asset. YouTube is the traffic engine.

---

## Two Layers (v1.6)

The plugin is split into two layers with a formal interface between them. This is the whole
point of the v1.6 architecture: playlist strategy is decided once, per playlist; video
production runs many times, always inside that frame, and never re-decides strategy.

```
LAYER A — PLAYLIST STRATEGY                     LAYER B — PER-VIDEO EXECUTION
(runs once per playlist, + re-plans)            (runs once per video)
skills/strategy/ + skills/shared/               skills/execution/ + skills/shared/

  playlist-strategy-orchestrator                  video-production-orchestrator
    → agent-01 (Playlist Mode)          Video      → agent-02 (Execution Mode)
    → agent-02 (Playlist Keyword Mode)  Brief      → agent-03 → PRODUCTION CORE
    → emits PlaylistStrategy doc        Card       → agent-04 → agent-05
      + one Brief Card per video   ───────────►    → post-publish: 03c / 05b / 06 / 07 / 09
```

**The interface is the Video Brief Card** (`references/layer-interface.md`). Layer A writes it;
Layer B reads it and never changes its `topic`, `angle`, `video_type` or `cta_lead_magnet` (only
advances `status`). The one exception is the keyword: the card's `primary_keyword` is a broad
playlist anchor, and Layer B always runs its own tighter video-specific keyword pass on top —
deliberate duplicate work, because the umbrella keyword is wide and the video keyword is narrow.
A one-off video with no playlist uses the `"ad-hoc video"` escape hatch.

## What's Inside

**15 Skills**, grouped by layer under `skills/`:

| Skill | Group | Layer | Trigger |
|---|---|---|---|
| `playlist-strategy-orchestrator` | strategy | A | "playlist strategy", "plan the playlist", "re-plan the playlist" |
| `agent-01-channel-strategist` | strategy | A | "what should I make next", "give me video ideas" |
| `agent-02-seo-title` | shared | A + B | "playlist keywords" (A) · "give me a title" (B) |
| `video-production-orchestrator` | execution | B | "new video", "new video cycle", "ad-hoc video" |
| `agent-03-script-writer` | execution | B | "write my script" |
| `agent-03c-shorts-producer` | execution | B (post) | "make the Shorts" |
| `agent-04-thumbnail` | execution | B | "thumbnail brief" |
| `agent-05-launch-publish` | execution | B | "ready to upload" |
| `agent-05b-multilingual-captions` | execution | B (post) | "translate captions" |
| `agent-06-analytics-reviewer` | execution | B (post) → feeds A | "review my analytics" |
| `agent-07-repurposing` | execution | B (post) | "repurpose" [R] / "community post" [C] |
| `strategic-launch-orchestrator` | launch | pointer | "teljes launch" → routes to the two jobs, chains nothing |
| `agent-09-conversion-bridge` | launch | — | "conversion audit", "CTA copy" |
| `channel-manager` | setup | — | "create new channel", "switch to channel X" |
| `agent-00-channel-foundation` | setup | — | "set up my channel", "channel foundation" |

**Architecture notes (v1.8):**
- Two layers, one plugin. The layer group dirs are registered via the `skills` array in `.claude-plugin/plugin.json`.
- The Video Brief Card is the interface between the two layers — schema and rules in `references/layer-interface.md`.
- Orchestrators read Drive ONCE and pass a CONTEXT BLOCK v2 (with `BRIEF_ID`) to every agent (`references/conventions.md`).
- **Cross-team contract (T1/T2/T3):** `playlist-strategy-orchestrator` writes `playlist_brief.json` and stops; `video-production-orchestrator` reads `optin_registry.json` written by T2. This plugin never invokes a T2 skill. See `layer-interface.md` §7.
- `strategic-launch-orchestrator` is a pointer only — it routes the operator to the traffic job (`playlist strategy`) and the funnel job (funnel skill (`{integrations.funnel_skill}`)) and chains nothing.
- Funnel building (lead magnet, landing page, email automation) is entirely team T2's — platform-agnostic contract, GHL today.

---

## How It Works

### 1. Multi-Channel Architecture
You manage multiple YouTube channels. The plugin keeps them strictly separated:

```
YT_Flywheel/                    ← master folder in your Google Drive
├── _Active_Channel.gdoc        ← active channel pointer (auto-managed)
├── _Channels_Registry.gsheet   ← all your channels (auto-managed)
└── [Channel Name]/             ← one folder per channel
    ├── 00_Foundation/
    ├── 01_Strategy/
    ├── 02_SEO_Titles/
    ├── 03_Scripts/
    ├── 04_Thumbnail_Briefs/
    ├── 05_Publish_Checklists/
    ├── 06_Analytics_Reviews/
    ├── 07_Repurposing_Briefs/
    ├── 08_Community/
    └── 09_Lead_Magnets/
```

When you say "switch to AI Agents Channel", every subsequent agent saves outputs to that channel's folder.

### 2. Knowledge Base
All agents read from the **YouTube Evergreen Flywheel Playbook v2.0** Google Doc:
- Doc ID: `playbook.doc_id` in your config
- Update the Playbook → all agents instantly use the new version (no plugin update needed)

### 3. The Workflow

**Setup (per channel, once):**
```
"create new channel called [name]"  →  channel-manager creates folders
"set up my channel"                  →  Agent 00 builds Foundation doc
```

**Playlist launch = two separate jobs (not chained here):**
```
Job 1 — TRAFFIC:  "playlist strategy"  →  playlist-strategy-orchestrator (Layer A)
   → PlaylistStrategy doc + N Video Brief Cards + playlist_brief.json, then stop
Job 2 — FUNNEL:   "start" / "lead magnet"  →  funnel skill from the config (T2)
   → reads playlist_brief.json, builds lead magnet + landing page, writes optin_registry.json
```
`video-production-orchestrator` reads `optin_registry.json` automatically once T2 is done.

**Every video — Layer B (PRE-publishing):**
```
"new video"  →  load next planned Brief Card → 02 Title/Desc → 03 Script → 04 Thumbnail → 05 Publish
```
No strategy step. The Brief Card carries topic, angle, video type, CTA and a broad anchor
keyword; Agent 02 adds the tight video-specific keyword.

**Five video types** (priority order, most→least used): `avatar_local` (primary — local build +
avatar clip), `avatar_native` (short-form / Shorts only, <2.5 min), `screen_recording`,
`talking_head` (real selfie recording, standalone or B-roll), `faceless_generated` (no avatar or
voice required). The avatar and voice are agnostic — the channel's own, a standard HeyGen
avatar, or a client's uploaded one — read from the Foundation doc, never hardcoded.
One-off video with no playlist: `"ad-hoc video"`.

**Every video — POST-publishing:**
```
06 Analytics → 07 Post-Publish Content ([R]+[C]) → 09 Conversion Audit
06's CONTENT DIRECTION UPDATE feeds Layer A → "re-plan the playlist" regenerates the planned Brief Cards
```

---

## Installation

### Cowork
Drop the `.plugin` file into Cowork's plugin install dialog. All 15 skills register
automatically (the `skills` array in `.claude-plugin/plugin.json` registers the layer group
dirs — `skills/strategy`, `skills/shared`, `skills/execution`, `skills/launch`, `skills/setup`).

### Manual (alternative)
Copy the contents of each `skills/<group>/` directory into your Cowork skills folder:
- Mac: `~/Library/Application Support/Claude/skills/`
- Windows: `%APPDATA%\Claude\skills\`

Then copy `CLAUDE.md` into Cowork → Settings → Global Instructions.

---

## Prerequisites

- Google Drive connector active in Cowork (the plugin saves all outputs there)
- Access to the Playbook Google Doc (or your own copy of it)

---

## Core Frame — Non-Negotiable

- The owner is a **Business Owner**, not a YouTuber
- YouTube is a traffic engine → email list is the real business asset
- Platform-independent: never name specific CRM tools, email platforms, or paid software in video scripts or descriptions
- No fake scarcity, no hustle culture, no viral chasing
- Goal: evergreen, consistent, qualified leads — not views, subscribers, or fame

---

## First Run

After install, just say:
```
"Set up Flywheel"
```
The channel-manager skill creates the YT_Flywheel master folder in your Drive, then guides you to create your first channel. From there, you're in the cycle.

---

**Version:** 1.8.0
**Author:** Zoltan Gero (humorketing@gmail.com)
**License:** Personal use
