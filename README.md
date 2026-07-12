# YouTube Evergreen Flywheel — AI Agent Team (Cowork Plugin)

A complete AI Agent Team that runs your full YouTube publishing workflow across **multiple channels**, with all outputs saved directly to **Google Drive**.

Built for Business Owners (not YouTubers). The email list is the asset. YouTube is the traffic engine.

---

## What's Inside

**13 Skills** (2 orchestrators + channel manager + 10 agents):

| Skill | Phase | Trigger |
|---|---|---|
| `strategic-launch-orchestrator` | Once per playlist | "playlist launch", "új playlist" |
| `video-production-orchestrator` | Every video | "új videó", "new video cycle" |
| `channel-manager` | Setup / switching | "create new channel", "switch to channel X" |
| `agent-00-channel-foundation` | Once per channel | "set up my channel", "channel foundation" |
| `agent-01-channel-strategist` | Pre | "what should I make next" |
| `agent-02-seo-title` | Pre | "give me a title" |
| `agent-03-script-writer` | Pre | "write my script" |
| `agent-03c-shorts-producer` | Post | "shortokat is kérem" |
| `agent-04-thumbnail` | Pre | "thumbnail brief" |
| `agent-05-launch-publish` | Pre | "ready to upload" |
| `agent-05b-multilingual-captions` | Post | "felirat fordítás" |
| `agent-06-analytics-reviewer` | Post | "review my analytics" |
| `agent-07-repurposing` | Post | "repurpose" [R] / "community post" [C] — two modes |
| `agent-09-conversion-bridge` | Post | "conversion audit", "CTA copy" |

**Architecture notes (v1.4):**
- Orchestrators read Drive ONCE and pass a CONTEXT BLOCK to every agent (`references/conventions.md`).
- Funnel building (lead magnet, landing page, GHL) is DELEGATED to the `email-list-building` plugin.
- Agent 08 merged into Agent 07 (Community mode). Agent 03b merged into the video orchestrator
  as an ElevenLabs API fallback. Legacy pipeline-orchestrator removed.

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
- Doc ID: `1tl712hBu4OARg_zrA5wu4kqtOcn729ssrqD1yEdmcLI`
- Update the Playbook → all agents instantly use the new version (no plugin update needed)

### 3. The Workflow

**Setup (per channel, once):**
```
"create new channel called [name]"  →  channel-manager creates folders
"set up my channel"                  →  Agent 00 builds Foundation doc
```

**Every video — PRE-publishing:**
```
01 Strategy → 02 SEO/Title → 03 Script → 04 Thumbnail → 05 Publish
```

**Every video — POST-publishing:**
```
06 Analytics → 07 Post-Publish Content ([R]+[C]) → 09 Conversion Audit → back to 01
```

---

## Installation

### Cowork
Drop the `.plugin` file into Cowork's plugin install dialog. All 11 skills register automatically.

### Manual (alternative)
Copy the `skills/` directory contents to your Cowork skills folder:
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

**Version:** 1.4.0
**Author:** Zoltan Gero (humorketing@gmail.com)
**License:** Personal use
