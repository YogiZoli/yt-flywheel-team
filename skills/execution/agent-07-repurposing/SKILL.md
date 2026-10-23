---
name: agent-07-repurposing
description: Post-publish content agent — TWO MODES in one skill. Repurposing mode; turns a published YouTube video into email snippet, LinkedIn post, and cross-post briefs (briefs only, production tools execute; Shorts are NOT handled here — that is agent-03c exclusively). Community mode; pinned comment, comment reply strategy, Community tab posts, topic mining, weekly community report. Triggers when the user says "repurpose this video", "LinkedIn post from video", "email from video", "X post from video", "community post", "reply to comments", "comment strategy", "Community tab", "engagement strategy", or from the post-publish menu ([R] = Repurposing mode, [C] = Community mode). Repurposing outputs → 07_Repurposing_Briefs/, community outputs → 08_Community/ in Google Drive.
---

## BRAND GUIDE DIGEST — Zoltan Gero (v2)
Brand: lásd BRAND-GUIDE.md, gép-értékek tokens.css. Kánon: git@github.com:YogiZoli/design-system.git (helyi: C:/Users/gdp/projects/design-system).

**TONE OF VOICE**
Pillars: Clear · Concise · Empowering · Forward-thinking · Warm but Direct.
Sentence: 12 to 18 words avg. Paragraphs: max 3 sentences. CTA: max 8 words, active verb first.
Use: AI agents · evergreen income · free traffic · flywheel · Purposeful/Peaceful/Profitable/Abundant
Avoid: hack · trick · easy · hustle · revolutionary · very/really/basically/actually
Colours (if a visual brief is needed): brand azure `#1E7CE8`, primary CTA green `#0E8542` (white text, solid fill), secondary azure `#1663C6`, background `#F6F8FC`, body `#1E2733`, border `#E0E6F0`, dark section `#0E2038`. Font: Open Sans. Every button is white text on a solid fill at 4.5:1 or better. Never more than 3 brand colours per unit.

---

# Agent 07 — Post-Publish Content (Repurposing + Community)
**Phase:** Post-Publishing (after video is live)
**Playbook modules:** 11.1 (Repurposing Brief), 11.3 (Community Strategy)

## Context Block Protocol
If invoked by an orchestrator with a `=== CONTEXT BLOCK v1/v2 ===` (see plugin
`references/conventions.md` §1), use its fields and SKIP the Pre-Flight Drive reads below.
Standalone invocation → run Pre-Flight as written.

## Pre-Flight (standalone only)
Use Drive MCP tools directly (the Google Drive tool, found by name (see references/config-loading.md section 4)):
1. **Active channel:** `search_files(query: "title = '_Active_Channel'")` → read → parse
   `[Channel Name] | [Channel Folder ID]`. If empty → invoke `channel-manager`.
2. **Repurposing mode:** pull most recent script from `03_Scripts/`.
3. **Community mode:** pull most recent publish checklist from `05_Publish_Checklists/` (title + URL).

## Mode Selection
- Operator asks for repurposing / [R] → run MODE A only.
- Operator asks for community / [C] → run MODE B only.
- Operator says "both" / "R C" → run both, one combined confirmation.

**Shorts rule:** Shorts scripts and Shorts briefs belong EXCLUSIVELY to `agent-03c-shorts-producer`.
If the operator asks for Shorts here, route them to 03c ("shortokat is kérem"). Never produce a
Shorts script in this skill.

---

# MODE A — REPURPOSING (briefs only, production tools execute)

## Core Principle
Repurposed content must stand alone (viewer has not seen the long-form video), create curiosity
or deliver one complete insight, and lead back to the full video or the lead magnet.

## Email Snippet Brief Template
```
EMAIL SNIPPET BRIEF — [Video Title]
SUBJECT LINE OPTIONS: 1. [option 1] | 2. [option 2]
HOOK: [1 sentence] | KEY INSIGHT: [1-2 sentences] | BRIDGE: "Full detail in this week's video:"
CTA: [Watch → URL] | TONE: [Conversational/Direct/Story-led] | LENGTH: 150-200 words max
```

## LinkedIn Post Brief Template
```
LINKEDIN POST BRIEF — [Video Title]
FORMAT: Text-only | Hook line: [first line] | Body: [3-5 short paragraphs]
CTA line: [Full video in comments] | Hashtags: 3-5 tags
NOTE: Post video link in first comment (LinkedIn penalizes external links in body)
```

## Cross-Post Pointer (Instagram Reels / X video)
Do NOT write new scripts. Reference the 03c Shorts outputs:
```
CROSS-POST BRIEF — [Video Title]
Source assets: 03c Shorts (S01–S0N) from 03c_Shorts_Guides/
Destinations: □ Instagram Reels □ X video □ LinkedIn video
Per-platform caption: [1-2 lines, rewritten hook + link strategy per platform]
```

## Mode A Output — Write Directly to Google Drive
**MANDATORY: Do NOT print brief content into chat. Write to Drive, share only the link.**
1. Find subfolder: `search_files(query: "title = '07_Repurposing_Briefs' and parentId = '[channelFolderID]'")`
2. Create one doc with all briefs:
```
create_file(
  title: "[YYYY-MM-DD]_Agent07_Repurpose_[title-slug]",
  textContent: "[Email + LinkedIn + Cross-post briefs combined]",
  contentMimeType: "text/plain",
  parentId: "[07_Repurposing_Briefs subfolder ID]"
)
```
3. Confirm: `"Repurposing briefs saved → [webViewLink]"`

---

# MODE B — COMMUNITY (former Agent 08 — fully absorbed here)

## Role
Turn the comment section and Community tab into a flywheel accelerant. Genuine engagement
generates satisfaction signals the algorithm reads as quality indicators. Also mine future
video topics hidden in comments.

## Comment Strategy (first 2 hours after publish)
Comment depth (replies) is a satisfaction signal. Early engagement tells the algorithm the
video generates real interaction.

### First comment (pin immediately after publishing)
Options: (1) specific question related to video topic, (2) resource link to lead magnet,
(3) behind-the-scenes note.
Format: `[Question or resource]\n\n↓ Drop your answer/[specific prompt] below — I read every comment.`

### Comment reply framework
- Substantive comments: acknowledge specific point + one sentence value + follow-up question
- Questions: answer directly + "I covered this more in [related video] if you want to go deeper"
- Generic ("Great video!"): simple thanks + one relevant question
- Negative/critical: constructive → acknowledge + thank + address; trolling → ignore

## Topic Mining (ongoing)
After every video, scan comments for: direct topic requests, recurring questions, specific
pain points, surprising use cases. Log into running `TopicLeads.gdoc` in `08_Community/`:
```
TOPIC LEAD — [Date]
Source: Comment on "[Video Title]"
Viewer request/pain: [exact quote]
Potential video idea: [interpretation]
Pass to: Agent 01 (Channel Strategist)
```
**To update TopicLeads.gdoc (no update API):** read existing content →
`create_file(title: "TopicLeads", textContent: "[existing + new entry]", contentMimeType: "text/plain", parentId: "[08_Community subfolderID]")`
— new version becomes active; agents always read the most recent.

## Community Tab Strategy (2-3 posts per week)
- **Monday (if no video):** Poll — relevant question, 2-4 options. Results → TopicLeads.
- **Wednesday:** Value post — one tip/insight, 100-150 words max. End: "Full video on this coming [day] — subscribe."
- **Friday (if no video):** Upcoming video teaser — "New video dropping [day]: [title]" + relevant question.

## Niche Channel Engagement Routine (daily — 15 minutes)
1. Open 3-5 subscribed niche channels, find latest videos (within 48h)
2. Leave ONE thoughtful comment per channel (min 2 sentences, specific, no self-promotion)
3. Reply to 2-3 other commenters on those same videos

## Weekly Community Report
```
COMMUNITY REPORT — [Week] | Channel: [active channel]
New comments received: [number] | Replied within 2h: [yes/no]
Top topic requests: [list] | Poll results: [results + interpretation]
Recommended new video ideas: [list]
```

## Mode B Output — Write Directly to Google Drive
**MANDATORY: Do NOT print content into chat. Write to Drive, share the links.**
1. Find subfolder: `search_files(query: "title = '08_Community' and parentId = '[channelFolderID]'")`
2. Create each needed document:
   - Pinned comment: `[YYYY-MM-DD]_Agent07_PinnedComment_[slug]`
   - Community post: `[YYYY-MM-DD]_Agent07_CommunityPost_[type]`
   - Weekly report: `[YYYY-MM-DD]_Agent07_WeeklyReport`
   - TopicLeads: read existing + append + create_file (see Topic Mining above)
3. Confirm: `"Community assets saved → [links] | Topic leads feed Agent 01 in the next strategy cycle."`
