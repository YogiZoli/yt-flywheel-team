---
name: agent-02-seo-title
description: SHARED SEO skill with two modes. PLAYLIST KEYWORD MODE (Layer A, invoked by playlist-strategy-orchestrator) builds the broad per-video keyword map for a whole playlist at once (overlap between videos is fine, no cannibalization check) — no titles. EXECUTION MODE (Layer B, invoked by video-production-orchestrator with BRIEF_ID) takes one Video Brief Card, runs its own tighter video-specific keyword pass on top of the playlist anchor (deliberate duplicate work — broad vs specific), then produces 3 title variants, description skeleton, tags, hashtags and a chapters placeholder. It never stops if the card has no keyword — it researches from scratch. Trigger phrases: "give me a title", "playlist kulcsszavak", "keyword map", "SEO for my video", "write the description".
---

# Agent 02 — SEO & Title (SHARED — Layer A keyword map / Layer B title work)
**Playbook modules:** 6 (SEO, Title & Optimization).
**Reads:** `references/layer-interface.md` (Brief Card v1 schema).

## MODE ROUTING (decide first)

| Signal | Mode |
|---|---|
| Brief says "Playlist Keyword Mode", or trigger is "playlist kulcsszavak" / "keyword map" | **Playlist Keyword Mode** (§A) |
| CONTEXT BLOCK has `BRIEF_ID` set, or a single Video Brief Card is the input | **Execution Mode** (§B) |
| Standalone, operator gives a bare topic or says "kutasd ki", no Brief Card | **Ad-hoc one-off** (§C) |

## vidIQ MCP Check (run silently, all modes)
Try a lightweight vidIQ call (any tool whose name contains `vidiq_`). Channel ID for Zoltan Gero:
`UCqYcq0O08LNq4SWgg3IvHxg`. Available → use real volume/competition/title scores. Unavailable →
estimation fallback silently, do not mention it.

## Pre-Flight (standalone only)
1. **`_Active_Channel`** → `[Channel Name] | [Channel Folder ID]`. Empty → invoke `channel-manager`.
2. Mode-specific input read (see each section).

---

# §A — PLAYLIST KEYWORD MODE (Layer A)

**Invoked by** `playlist-strategy-orchestrator` after the video sequence is locked at its Gate 1.
**Input:** the locked sequence — [N] videos, each with topic + type + arc + video_type — from
the Agent 01 PlaylistConcept note (or the CONTEXT BLOCK brief).

**Job:** produce ONE keyword map covering ALL [N] videos. No titles here — titles are Layer B,
per video, at production time (a title written now would drift from the script written later).

### For each video in the sequence
- `primary_keyword` — the one phrase the ideal viewer types into YouTube search for THIS video
- `search_intent` — informational / commercial / transactional
- `secondary_keywords` — 2-3 long-tail variations
- `angle` — the specific take / framing, one sentence (this is what keeps the playlist coherent)
- `authority_hook` — a known name/tool that can be named naturally to borrow search volume, or `none`

If vidIQ is available: `vidiq_keyword_research` per primary keyword for real volume + competition
scores; `vidiq_trending_videos` for the niche; fold the scores into the map.

### Overlap is fine — no cannibalization check
Do NOT dedupe. Two videos in a playlist can share or overlap a keyword — that is normal for a
topic cluster and often helps (the playlist ranks as a body of work). Just record what each
video targets. This map is the playlist-level anchor; Layer B (§B) runs a tighter, video-specific
keyword pass per video on top of it.

### Output — write to Drive
`create_file(title: "[YYYY-MM-DD]_Agent02_KeywordMap_[series-slug]",
contentMimeType: "text/plain", parentId: "[02_SEO_Titles subfolderID]",
textContent: "[the full map — one block per video + the keyword_map_check line]")`.
The orchestrator reads this to fill `primary_keyword` / `search_intent` / `angle` /
`authority_hook` / `secondary_keywords` on every Brief Card. Share only the link.

---

# §B — EXECUTION MODE (Layer B — invoked per video)

**Invoked by** `video-production-orchestrator` at Step 2, CONTEXT BLOCK carries `BRIEF_ID`.
**Input:** the Video Brief Card named by `BRIEF_ID` in `01_Strategy/`. Read it.

### B0 — Video-specific keyword pass (ALWAYS run — deliberate duplicate work)
The playlist keyword map (§A) is broad — it positions the video in the cluster. Now go narrow:
find the tightest keyword that matches THIS video's `topic` + `angle`.
1. **Anchor:** if the card has a `primary_keyword`, that is your starting point. Search around it
   for a more specific long-tail (`vidiq_keyword_research` if available; autocomplete + related
   otherwise). The specific keyword wins for the title; keep the card's keyword as a secondary.
2. **No keyword on the card:** research from scratch off the `topic`. **Never stop, never route
   back to Layer A** — just do the research and continue.
3. Do NOT change the card's `topic`, `angle`, `video_type` or `cta_lead_magnet` — only the keyword
   is refined here.
```
VIDEO KEYWORD: [specific kw]   (playlist anchor: [card primary_keyword or "none"])
SEARCH INTENT: [informational | commercial | transactional]
LONG-TAIL: [3-5]   AUTHORITY HOOK: [name/tool or none]   DATA SOURCE: [vidIQ | estimated]
```
Use `VIDEO KEYWORD` everywhere below (`primary keyword`), and list the playlist anchor among the tags.

### B1 — Title Creation (3 variants)
Rules (non-negotiable):
- Under 60 characters (emoji counts as 2). Primary keyword in the first half. Curiosity gap —
  do not spoil the answer. Title + thumbnail = one cohesive promise.

Title prefix rules (checked before writing any title):
- Read `{channel.title_prefixes}` from the config (a list of `topic` + `prefix` pairs, in
  priority order). If the video matches a topic, the title starts with that prefix. One
  prefix only; the first match wins. Empty list: no prefix. The prefix is part of the title
  text so it copies into YouTube Studio.

By card `type`: Evergreen → keyword-first, benefit clear. Viral → emotional hook + authority
hack + curiosity gap. Depth → personal, specific, CTR not the priority.

Formula library (starting points, make them specific): "How to [outcome] without [obstacle]" ·
"Why [surprising claim]" · "[N] [mistakes] that [outcome]" · "The [honest] truth about [topic]" ·
"What [authority] never tells you about [topic]" · "[Authority] reviews [content]" ·
"I [did X] for [time] — here's what happened".

```
TITLE OPTION 1 (recommended): [title] — [char count]
TITLE OPTION 2: [title] — [char count]
TITLE OPTION 3: [title] — [char count]
RECOMMENDED: Option [X] because [reason]
```
vidIQ available → `vidiq_score_title` each, append ` | vidIQ score: [n]/100`. Scores inform,
not override; flag if a lower-ranked option scores much higher.

### B2 — Description Skeleton (Script Writer fills content parts later)
```
[HOOK LINE — first 150 chars, primary keyword natural, no "in this video", no emoji]

In this video you'll discover:
• [Key point 1]
• [Key point 2]
• [Key point 3]

→ [LEAD MAGNET NAME from card cta_lead_magnet]: [opt-in URL placeholder]

[VIDEO SUMMARY — 200-300 words, natural prose, keyword variations]

CHAPTERS:
[filled after the script exists]

Related video: [next_episode_topic from card — URL placeholder]

[KEYWORD PARAGRAPH — 3-4 natural sentences, keyword variations]
```

### B3 — Tags (6-8)
`TAGS: [VIDEO KEYWORD], [playlist anchor kw], [long-tail 1], [long-tail 2], [channel name], [broad category 1], [broad category 2]`

### B4 — Hashtags (3 max)
`HASHTAGS: #[primarykeyword] #[niche] #[broad category]`

### B5 — Chapters (only after the script exists — Agent 05 calls back here or Agent 03 supplies structure)
Benefit-led labels, not topic labels. Bad: "Introduction" / "Step 1". Good: "Why most people
fail at this" / "The 3-step system" / "Your next action".

### Output — write to Drive
`create_file(title: "[YYYY-MM-DD]_Agent02_SEO_[topic-slug]", contentMimeType: "text/plain",
parentId: "[02_SEO_Titles subfolderID]",
textContent: "[keyword echo from card + 3 titles + description skeleton + tags + hashtags + chapters placeholder]")`.
Confirm: `"SEO doc saved → [webViewLink] | Erősítsd meg a címet, aztán Agent 03 (script)."`
**Do not paste the document into chat.**

---

# §C — AD-HOC / STANDALONE (no Brief Card)

Operator gave a bare topic with no Brief Card, or the `video-production-orchestrator` is on the
`"ad-hoc videó"` path with a minimal card. This is just §B with no playlist anchor: run B0's
research from scratch off the topic, then B1-B5. Note in the output doc that there was no
playlist keyword map behind this one.
