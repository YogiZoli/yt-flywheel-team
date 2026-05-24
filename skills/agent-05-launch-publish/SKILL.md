---
name: agent-05-launch-publish
description: Use this agent when ready to upload a video to YouTube. Triggers when the user says "ready to upload", "publish my video", "upload checklist", "launch my channel", "60 day plan", or "6 week plan". Also handles end screen, Cards, captions checklist, HeyGen Production Prompt, Publishing Artifact with copy buttons, and chapter finalization. Phase 2 triggers on "HeyGen jóváhagyva" or equivalent to finalize chapter timestamps. Outputs save to the active channel's 05_Publish_Checklists/ folder in Google Drive.
---

# Agent 05 — Launch & Publish
**Phase:** Pre-Publishing + HeyGen Production (two-phase workflow)
**Playbook modules:** 6.3 (Captions), 9 (Launch Plan)

## Pre-Flight: Active Channel Check
Use Drive MCP tools directly — do not ask the operator to provide these:
1. **Read `_Active_Channel.gdoc`:** `search_files(query: "title = '_Active_Channel'")` → take first result's `id` → `read_file_content(fileId)` → parse line 1: `[Channel Name] | [Channel Folder ID]`. Store **Channel Folder ID**. If empty → invoke `channel-manager`.
2. **Pull latest assets automatically:**
   - Title + SEO data: `search_files(query: "title = '02_SEO_Titles' and parentId = '[channelFolderID]'")` → subfolder → read most recent (extract title, tags, hashtags, description hook)
   - **Script (CRITICAL — read full text):** `search_files(query: "title = '03_Scripts' and parentId = '[channelFolderID]'")` → subfolder ID → `search_files(query: "parentId = '[subfolderID]'")` → read most recent → **store complete script text for chapter generation and HeyGen conversion**
   - Thumbnail brief: `search_files(query: "title = '04_Thumbnail_Briefs' and parentId = '[channelFolderID]'")` → read most recent
   - Lead magnet: `search_files(query: "title = '09_Lead_Magnets' and parentId = '[channelFolderID]'")` → read most recent → extract lead magnet name and link

## Your Role
**Two-phase workflow.** Phase 1 (runs now): prepare all metadata, create Publishing Artifact with copy buttons, generate HeyGen Production Prompt. Phase 2 (triggered after HeyGen video approved): finalize chapter timestamps, update same artifact. HeyGen video generation runs LAST — after thumbnail and all publishing copy are ready.

---
## PHASE 1: Publishing Prep + HeyGen Trigger

### Step 1: Generate YouTube Description
Structure:
- **Opening paragraph** (lines 1-3): hook that works as standalone preview. First 150 characters are critical — must compel click without "in this video" or emojis. Match the title's promise exactly.
- **Lead magnet line:** `→ [Lead Magnet Name]: [link]` — always second, before anything else
- **Chapters section:** `CHAPTERS` header + draft timestamps (generated in Step 3 below)
- **About paragraph:** 2-3 sentences about Zoltan + channel subscribe CTA
- **3 hashtags:** at the very end of description (use exact hashtags from Agent 02)

### Step 2: Generate Tags
**MAXIMIZE to ~500 characters** — YouTube's hard limit. Do NOT use only 6-8 tags; fill the full allowance. Combine: primary keyword, long-tail variations, location-specific phrases, broader category terms, year (e.g. "cambodia 2025"), channel/brand name (Zoltan Gero), synonyms, and alternative phrasings. Count characters as you build the list and stop just under 500. The operator pastes all tags at once into YouTube Studio (deletes old tags → Ctrl+V → done). Always display the char count in the artifact badge (e.g. "498 / 500").

### Step 3: Generate Draft Chapters
Derive chapter timestamps directly from the script sections:
1. Identify all script sections: [HOOK], [OUTCOME], [TESTIMONIAL], [SUBSCRIBE CTA], [STEP X] titles, [ENGAGEMENT CTA], [CONVERSION CTA], [END SCREEN]
2. Count approximate words per section
3. Calculate cumulative timestamps assuming **130 words per minute** narration pace (standard YouTube teaching pace)
4. Map to chapter titles:
   - [HOOK] + [OUTCOME] + [TESTIMONIAL] → "0:00 Introduction"
   - Each [STEP X: Title] → timestamp + step title (simplified if too long)
   - [CONVERSION CTA] → "Next Steps"
   - Skip [SUBSCRIBE CTA], [ENGAGEMENT CTA], [END SCREEN] as separate chapters (fold into nearest section)
5. Format as YouTube chapter markers (one per line, minimum 10-second gaps between chapters):
```
0:00 Introduction
0:45 [Step 1 title]
2:30 [Step 2 title]
...
8:10 Next Steps
```
6. Label clearly as **[DRAFT — verify after watching approved HeyGen video]**

### Step 4: Pre-Upload Checklist

**Title & Metadata**
- [ ] Title confirmed from Agent 02 (under 60 characters, keyword in first half)
- [ ] Description complete (hook in first 150 chars, lead magnet link first, draft chapters)
- [ ] Tags entered (6-8 maximum)
- [ ] 3 hashtags in description
- [ ] Language set to English (or primary audience language)
- [ ] Category selected (Education, How-to & Style, Science & Tech — match niche)

**Thumbnail**
- [ ] Thumbnail created from Agent 04 brief (via Nano Banana 2 — attach profile photo as reference)
- [ ] Thumbnail exported from Canva and saved to Google Drive (04_Thumbnail_Briefs folder)
- [ ] Thumbnail Drive link added to Publishing Artifact
- [ ] Thumbnail checked at small size (mobile view — 120×68px minimum readable)
- [ ] A/B variant ready in YouTube Studio (if running test)

**Timing**
- [ ] Scheduled for consistent publish day and time
- [ ] Peak audience time confirmed (YouTube Analytics → Audience → When your viewers are on YouTube)
- [ ] Minimum 48h gap from previous video (new channels)

**End Screen (last 20 seconds)**
- [ ] End screen added in YouTube Studio after upload
- [ ] End Screen 1: Subscribe button
- [ ] End Screen 2: Most recent video on channel (or next in playlist if it exists)
- [ ] Matches the gesture direction written in the script

**Cards (in-video)**
- [ ] Card 1 added at ~20% timestamp — Type: Playlist — Target: video's playlist
- [ ] Card 2 added at ~70% timestamp — Type: Video — Target: most recent channel video
- [ ] First Card no earlier than 20% into the video

**Settings**
- [ ] Not made for kids — set to "No, it's not made for kids"
- [ ] AI-generated content disclosure — checkbox ticked
- [ ] Visibility set to Unlisted until ready to publish

**Captions**
- [ ] HeyGen SRT file downloaded from HeyGen Studio after video approval
- [ ] SRT saved to Drive: `Captions/[title-slug]_ENG_cc_timestamps.srt`
- [ ] SRT uploaded to YouTube Studio → Subtitles → Add → Upload file → With timing → English
- [ ] Do NOT manually edit auto-generated captions — HeyGen SRT replaces them

**Playlist**
- [ ] Video added to correct playlist (from Agent 01 brief)
- [ ] Playlist order checked (newest near top)

### Step 5: Gather Card, End Screen & Thumbnail Data

Before creating the artifact, collect the following — do NOT ask the operator for this data, derive it automatically:

**Playlist name:** read from Agent 01 brief (already loaded in Pre-Flight).

**Estimated video length:** total script word count ÷ 130 wpm = total minutes. Convert to mm:ss.

**Cards — two cards required:**
- Card 1: Playlist card — target = the playlist this video belongs to (from Agent 01 brief). Timestamp = ~20% of estimated video length.
- Card 2: Video card — target = most recently published video on the channel. Use `mcp__bab7487b-ef77-4f71-a988-c5cfd8566b9a__vidiq_channel_videos` to get the channel's latest video title and URL. If this is the first video on the channel, set Card 2 target to BLANK and note it in the artifact.

**End Screen — two elements (last 20 seconds of video):**
- End Screen 1: Subscribe button (always present)
- End Screen 2: Video element — use the most recent video from VidIQ channel videos. If first video on channel, note "Add once a second video is published."

**Thumbnail Drive link:** Search `04_Thumbnail_Briefs` subfolder for the most recently created image file (jpg/png):
`search_files(query: "parentId = '[04_thumbnail_subfolder_id]'")` → most recent result → use its `webViewLink`.
If no image file found, use placeholder: `[Export thumbnail from Canva via Agent 04 — paste Drive link here]`

---

### Step 6: Create Publishing Artifact

Use `mcp__cowork__create_artifact` to create a persistent HTML page with all publishing copy and working copy buttons. This artifact lives in chat and can be reopened — it is the single source of truth for all upload copy.

**HTML requirements:**
- Clean layout: off-white background (#fafafa), dark text, subtle card-style sections
- Page header: "📋 Publishing Pack — [video title]"
- 10 content sections. Sections 1–4 and 6–9 each have:
  - Bold section label
  - Content in a `<pre>` or `<div>` block with monospace font and light grey background
  - **Copy button** — use a ROBUST two-method approach: try `navigator.clipboard.writeText()` first; if it fails or is unavailable, fall back to creating a hidden `<textarea>`, selecting its content, and calling `document.execCommand('copy')`. This ensures the button works in all environments without requiring the user to press Ctrl+C. Button briefly shows "✓ Copied!" (green) for 1.5 seconds after click, then resets to "Copy".
- Section 5 (SETTINGS) is a visual-only reminder block — amber/yellow tint, no copy button.
- **Sections in order:**
  1. **TITLE** — the YouTube video title. Apply emoji prefix rule (from Agent 02): 🇰🇭 for Cambodia videos, 🚀 for money/income/bootstrapping videos. The emoji must be part of the title text so it copies correctly.
  2. **DESCRIPTION** — full description text including chapters (chapters show [DRAFT] inline)
  3. **TAGS** — comma-separated list, paste-ready for YouTube Studio. Display a live character counter badge next to the section label (e.g. "498 / 500" in green). Tags MUST be maximized to ~500 characters. Operator workflow: delete all existing tags in YouTube Studio → paste all tags with one Ctrl+V.
  4. **HASHTAGS** — 3 hashtags on one line
  5. **SETTINGS** — styled as a checklist reminder block (amber/yellow background tint, no copy button):
     ```
     ☑ Not made for kids — select "No, it's not made for kids"
     ☑ AI-generated content disclosure — check the AI content box
     ☑ Visibility — set to Unlisted until ready to publish
     ```
  6. **PLAYLIST** — exact playlist name to add this video to (paste into YouTube Studio playlist field)
  7. **CARDS** — paste-ready reference block (added in YouTube Studio → Video Details → Cards):
     ```
     Card 1 | At: [~20% timestamp] | Type: Playlist | Target: [playlist name]
     Card 2 | At: [~70% timestamp] | Type: Video   | Target: [most recent video title — or BLANK if first video]
     ```
  8. **END SCREEN** — paste-ready reference block (last 20 seconds; added in YouTube Studio → Video Details → End screen):
     ```
     End Screen 1 | Type: Subscribe
     End Screen 2 | Type: Video | Target: [most recent video title — or "Add when 2nd video is published"]
     ```
  9. **THUMBNAIL** — Google Drive link to thumbnail image exported from Canva (Agent 04 output). Display as a clickable link inside the copy block. If placeholder, show the placeholder text in italics with a distinct orange border on the block.
  10. **CHAPTERS (DRAFT)** — timestamps only, with a visible amber badge `[DRAFT]` next to the section header
  11. **DESIGN BRIEF** — the full thumbnail brief from Agent 04 (read from `04_Thumbnail_Briefs` subfolder). Styled with a distinct purple/lavender border and background so it stands out. Scrollable if long (max-height ~320px with overflow-y: auto). Has its own Copy button. Label: "🎨 Design Brief" with sub-label "Nano Banana / Canva". This allows the operator to copy the full design brief instantly without searching Google Drive.

  12. **POST-PUBLISH ACTIONS** — shown at the bottom of the artifact after the video is published. Dark background section (#111), header "🚀 Post-Publish Menu", subtext "All optional — run in any order after the video is live." Five action buttons in a row, each styled as a pill/chip. On click, each button copies a trigger phrase to clipboard so the operator can paste it into chat:
      - **[S] Shorts** → copies `"shortokat is kérem"` | grey chip
      - **[R] Repurpose** → copies `"repurpose"` | grey chip
      - **[C] Community** → copies `"community post"` | grey chip
      - **[E] Email/Funnel** → copies `"conversion audit"` | grey chip
      - **[M] Captions** → copies `"felirat fordítás"` | green chip (highest SEO ROI — visually distinguished)
      Below the buttons: small text — "💡 [M] Captions = 25 language SRT upload. Free. ~3 min. Biggest SEO boost."

- Footer: small grey text — "Chapters are estimates (130 wpm). After HeyGen video approval, say 'HeyGen jóváhagyva' to finalize."
- **Save the artifact `id`** from the tool response — required for Phase 2 update.

### Step 7: Generate HeyGen Production Prompt

After the Publishing Artifact is live, convert the script (fetched in Pre-Flight) into HeyGen marker format. Save to Drive.

#### Standing HeyGen Settings (v4 — Zoltan Gero)

**Avatar:**
- Avatar ID: `bfcee432248442539191df03f922f575`
- Frame: Circle Frame applied throughout
- Engine: Motion Engine Avatar III, Unlimited Engine
- Fallback: Do NOT substitute silently. Flag immediately and wait for instruction.

**Voice:**
- Voice ID: `99b6f9eca33e474483987860185fdc43`
- Speed: Minimum +25% faster than default, recommended +50% faster. Mandatory, entire video, no exceptions.
- Fallback: substitute free young male voice, flag which.

**Recording Rules:**
- Every recording clip used ONCE only per video. No repeats, no loops, no "continued" markers.
- Each [RECORDING: X] = a unique physical clip the operator uploads.
- After a recording has been used once, subsequent coverage uses [BROLL: description].
- Label recordings sequentially: Recording_01, Recording_02, Recording_03, etc.

**Script Conversion (Agent 03 format → HeyGen marker format):**
- Each major section → `[CHAPTER: Title | Subtitle if applicable]`
- All spoken text → `[VO]` lines
- Key insights, CTAs, emotional hooks → precede with `[AVATAR]` (circle frame)
- Personal uploaded footage → `[RECORDING: Recording_XX, description]`
- Library B-roll → `[BROLL: description]`
- Avatar appears approx every 1-2 minutes, 1-2 sentences only each time
- Faceless-First philosophy: avatar is accent, not constant presence
- No colons or em dashes in any [VO] text (Global Script Style Rules from Agent 03)
- `[CALLOUT: phrase]` markers from Agent 03 → translate to `[TEXT OVERLAY: "phrase" | Style: Bold | Align: Center | Duration: 2.5s]` placed on the line immediately following the [VO] line it belongs to. Max 3 per video. These are branded keyword callouts — short, styled text overlays that highlight the key phrase of a sentence.

#### HeyGen Prompt Document Structure (always use this order)

```
# HeyGen Video Agent — Video Prompt
### For: Zoltan Gero | YouTube Teaching & Tutorial Videos
Template v4.0 | Changes: [brief note on what changed from previous version]

---
HOW TO USE THIS DOCUMENT:
1. Open HeyGen Video Agent
2. Start a new project
3. Paste this entire document as the first message
---
## CREDIT ESTIMATION, DO NOT SKIP
Estimate total video credits before generating.
---
## SECTION 1, TECHNICAL SETTINGS
- Orientation: Landscape (16:9)
- Target Video Length: 8-10 minutes, MINIMUM 8 minutes strictly enforced.
- Language: English
- Voice Speed: FASTER than default. Minimum +25% faster, recommended +50% faster. Mandatory, entire video.
---
## SECTION 2, AVATAR
- Avatar ID: bfcee432248442539191df03f922f575
- Frame: Circle Frame applied throughout every avatar appearance
- Engine: Motion Engine Avatar III, Unlimited Engine
- Fallback rule: Do NOT substitute silently. Flag immediately.
## SECTION 3, VOICE
- Voice ID: 99b6f9eca33e474483987860185fdc43
- Speed: +50% faster than default (minimum +25%). Applies to entire video. No exceptions.
- Fallback rule: substitute free young male voice, flag which one was used.
```

```
## SECTION 4, STYLE, LAYOUT & VISUAL PHILOSOPHY
Faceless-First. Avatar = accent, not constant presence. Avatar appears every 1-2 minutes, 1-2 sentences max. All other content = B-roll, recordings, or voiceover only.
## SECTION 5, VIDEO FOOTAGE & BACKGROUND
- All personal recordings used once only per video. No loops. No repeats.
- B-roll priority: (1) operator-uploaded footage (2) HeyGen library (3) text/slide overlay
## SECTION 6, CHAPTER OPENERS
Each [CHAPTER] marker triggers a visual chapter opener. Style: bold text on dark background, 3-5 seconds.
## SECTION 7, AVATAR APPEARANCE PATTERN
Circle Frame. Avatar precedes key insights, CTAs, and emotional hooks only. Max 1-2 sentences per appearance.
## SECTION 8, CTA STRUCTURE — Ownership Formula
1. Subscribe CTA: after Testimonial section
2. Engagement CTA: after most valuable step (mid-video)
3. Conversion CTA: near end, before end screen — direct to lead magnet
End card: final 3 seconds — channel name + subscribe + lead magnet link
---
## SECTION 9, VIDEO SCRIPT
[converted script with CHAPTER, AVATAR, VO, RECORDING, BROLL markers]
```

Save HeyGen prompt to Drive root (`parentId: 0AG9wJnPFjzalUk9PVA`).
File naming: `[YYYY-MM-DD]_HeyGen_[title-slug]_Video[ep-number]_v1`

**Phase 1 completion message to operator:**
`"✅ Publishing Artifact live in chat — all copy buttons ready | ✅ HeyGen prompt saved → [heygen-link] | Paste the HeyGen document into HeyGen Video Agent and start generation. Come back when you've watched and approved the final video: say 'HeyGen jóváhagyva' + any chapter timing corrections. | ✅ Checklist saved → [checklist-link]"`

---

## PHASE 2: Chapters Finalization (Post-HeyGen Approval)

**Trigger:** Operator says "HeyGen jóváhagyva", "approved", "chapters ok", "jóváhagytam", "mehet a chapters", or any equivalent confirmation the final HeyGen video has been watched and approved.

Steps:
1. Ask: "Any chapter timestamp corrections? Or should I confirm the draft estimates as-is?" (Keep it short — one question.)
2. Apply corrections if provided; otherwise use draft timestamps as confirmed
3. Generate final chapter list (clean, no DRAFT label)
4. **Update the Publishing Artifact** using `mcp__cowork__update_artifact` with the artifact `id` saved in Phase 1:
   - Replace CHAPTERS section content with finalized timestamps
   - Remove the amber [DRAFT] badge from the chapters header
   - Update the DESCRIPTION section to include the finalized chapters (replace the draft placeholder)
5. Confirm: `"✅ Chapters finalized. Publishing Artifact updated."`
6. **Automatically invoke Agent 05b** — do NOT wait for operator to type "felirat fordítás".
   Say: `"▶ Starting Agent 05b — Multilingual Captions (25 languages, free, ~3 min)..."`
   Then invoke `agent-05b-multilingual-captions` skill immediately.
   Pass: active channel folder ID, English SRT location (from Captions/ subfolder), video ID (ask once if not yet provided).
   Agent 05b runs to completion and prints its own summary report.

---

## Publish Day Protocol

### Before making public:
1. Like your own video
2. Pin your own comment: ask a specific question OR share the lead magnet link
3. Set reminder to reply to all comments within first 2 hours

### After publishing:
1. Copy video URL and share with known ideal viewers (personal message, not broadcast)
2. Add video link to email signature if not already present
3. Pass video URL to Agent 07 (Repurposing) and Agent 08 (Community)
4. **Multilingual captions (SEO boost):** Say "felirat fordítás" or "M" to trigger Agent 05b.
   Translates English SRT to 25 languages, uploads to YouTube. Free, ~3 min. Zero storage impact.

---
## 6-Week Launch Plan (new or relaunching channels only)

### Pre-Launch (Week 0)
- Complete Agent 00 checklist
- Produce 2-3 videos before launch day — never go live with just one
- Choose publish day and time — commit to it for minimum 12 weeks
- Set up lead magnet and opt-in page

### Week 1 — Day 1
- Publish first video with full checklist complete
- Like own video before making public
- Pin first comment
- Reply to every comment within 2 hours
- Personal outreach to known ideal viewers

### Daily Engagement Routine (Days 2-42)
Morning (15 minutes):
- Subscribe to 3-5 relevant channels in niche (turn on notifications)
- Reply to any new comments on own videos

Afternoon (15 minutes):
- Leave 3 thoughtful, specific comments on relevant videos in niche
- Engage with 5 other commenters on those same videos

Ongoing:
- If someone messages and matches ideal viewer profile → send channel link
- Review comments for future video topic ideas → pass to Agent 01

### Weekly Review Trigger
Every 7 days: pass data to Agent 06 (Analytics Reviewer)
Metrics to collect: Views per video, Average retention %, CTR % (by traffic source), New subscribers per video, Email opt-ins from YouTube, Comments received

---

## Captions Workflow

**Philosophy:** Always use the HeyGen-generated caption file — it is more accurate than YouTube auto-captions, especially for Zoltan's European accent, AI/tech vocabulary, and AI-cloned voice. Upload it as the English closed caption track on YouTube.

### Naming Convention
`[title-slug]_ENG_cc_timestamps.srt`
Example: `ai-agents-content-workflow_ENG_cc_timestamps.srt`

- `ENG` = language code
- `cc` = closed captions (primary, same language as audio)
- `timestamps` = file includes timing data
- Future translated tracks use `sub` instead of `cc`: `ai-agents-content-workflow_HUN_sub_timestamps.srt`

### Drive Folder Structure
Captions live in a dedicated subfolder inside the active channel folder:
```
[Channel Folder]/
  Captions/
    [title-slug]_ENG_cc_timestamps.srt
```

### Step-by-Step (runs in Phase 2, after HeyGen video approval)

1. **Remind operator to download HeyGen caption file:**
   > "In HeyGen Studio, open your finished video → go to Captions → download the SRT file."

2. **Save to Drive** using Drive MCP:
   - Find or create `Captions/` subfolder: `search_files(query: "title = 'Captions' and parentId = '[channelFolderID]'")`
   - If not found: `create_file(title: "Captions", mimeType: "application/vnd.google-apps.folder", parentId: "[channelFolderID]")`
   - Upload the SRT: `create_file(title: "[title-slug]_ENG_cc_timestamps.srt", textContent: "[SRT content]", contentMimeType: "text/plain", parentId: "[Captions subfolder ID]")`

3. **Upload to YouTube Studio:**
   - YouTube Studio → Videos → click video → Subtitles tab
   - Click "Add" → Upload file → Select "With timing"
   - Upload the `.srt` file → Language: English → Save
   - This replaces auto-generated captions — do NOT delete auto-generated first; upload overwrites it

4. **Confirm in Publishing Artifact checklist:**
   - [ ] HeyGen SRT downloaded
   - [ ] SRT saved to Drive: `Captions/[title-slug]_ENG_cc_timestamps.srt`
   - [ ] SRT uploaded to YouTube Studio as English closed captions

---

## Output — Write Directly to Google Drive

**MANDATORY: Do NOT print checklist content into chat. Write to Drive immediately, then share only the link.**

Execute these Drive MCP steps (tool prefix: `mcp__4dfc969f-edaf-4938-b2b2-5a86f07b939f`):

1. **Find subfolder:** `search_files(query: "title = '05_Publish_Checklists' and parentId = '[channelFolderID]'")`  → get subfolder ID

2. **Create document:**
```
create_file(
  title: "[YYYY-MM-DD]_Agent05_PublishChecklist_[title-slug]",
  textContent: "[completed checklist with all boxes checked / flagged items with reasons]",
  contentMimeType: "text/plain",
  parentId: "[05_Publish_Checklists subfolder ID]"
)
```

3. **Confirm to operator** using the `webViewLink` from the response — include in Phase 1 completion message above.
   After 7 days of data, run Agent 06 (Analytics Reviewer).
