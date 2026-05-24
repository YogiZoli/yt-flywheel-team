---
name: agent-05b-multilingual-captions
description: Post-publish multilingual caption generator. Translates English SRT to 25 languages using free Google Translate (no API key, no paid service), saves all SRTs to Drive, uploads to YouTube as subtitle tracks. Triggers after video is published. Phrases: "fordítsd le a feliratot", "multilingual captions", "05b", "caption fordítás", "felirat fordítás", "sub upload", "subtitle upload", "translate captions", "25 nyelv", "fordítás felirat".
---

# Agent 05b — Multilingual Caption Generator & Uploader
**Phase:** Post-publish (runs after Agent 05 Phase 2 is complete)
**Cost:** Zero — translation via free Python library (`deep-translator`), no API key, no paid service
**Token cost:** Minimal — Claude orchestrates only; translation runs in bash

## Overview
Translates the English SRT caption file (created by Agent 05) into 25 languages and uploads each
as a YouTube subtitle track. Result: video becomes discoverable via search in 25 languages.
This is the highest-ROI SEO action available after publishing — captions are indexed by YouTube's
search algorithm; audio tracks are not.

---

## Pre-Flight: Active Channel + Assets

Use Drive MCP tools directly — do not ask operator to provide these:

1. **Read `_Active_Channel.gdoc`:** `search_files(query: "title = '_Active_Channel'")` → first
   result → `read_file_content(fileId)` → parse line 1: `[Channel Name] | [Channel Folder ID]`.
   Store **Channel Folder ID**.

2. **Find the English SRT in Drive:**
   - Find Captions subfolder: `search_files(query: "title = 'Captions' and parentId = '[channelFolderID]'")`
   - Find English SRT: `search_files(query: "title contains '_ENG_cc_timestamps' and parentId = '[captionsSubfolderID]'")`
   - Read SRT content: `read_file_content(fileId: "[SRT file ID]")`
   - If not found: ask operator to paste the SRT text directly into chat, or download from
     HeyGen Studio (same file saved in Agent 05 Phase 2)

3. **Get video title slug:** Parse from the SRT filename (everything before `_ENG_cc_timestamps`)

4. **Get YouTube video ID:** Ask operator: "Kérem illessze be a YouTube videó URL-jét
   (pl. https://youtube.com/watch?v=XXXXXXXXXXX)" — extract the `v=` parameter.

---

## Step 1: Translate SRT to 25 Languages (Python, Free)

Run the following Python script in bash. It:
- Installs `deep-translator` automatically (free, uses Google Translate web, no API key)
- Parses English SRT (preserves all timestamps, translates text lines only)
- Outputs 25 translated SRT strings in JSON

**Replace `SRT_CONTENT` with the actual SRT text read from Drive in Pre-Flight.**

```python
import subprocess, sys
subprocess.run([sys.executable, "-m", "pip", "install",
    "deep-translator", "--break-system-packages", "-q"], check=True)

import re, time, json
from deep_translator import GoogleTranslator

SRT_CONTENT = """PASTE_SRT_CONTENT_HERE"""

# 25 target languages: (google_translate_code, youtube_bcp47_code, display_name)
LANGUAGES = [
    # Tier 1 — mandatory (highest YouTube audience + SE Asia priority)
    ("es",    "es",      "Spanish"),
    ("pt",    "pt-BR",   "Portuguese (Brazil)"),
    ("de",    "de",      "German"),
    ("fr",    "fr",      "French"),
    ("it",    "it",      "Italian"),
    ("vi",    "vi",      "Vietnamese"),
    ("th",    "th",      "Thai"),
    ("km",    "km",      "Khmer"),
    ("tl",    "fil",     "Filipino"),
    ("ms",    "ms",      "Malay"),
    # Tier 2 — recommended
    ("id",    "id",      "Indonesian"),
    ("nl",    "nl",      "Dutch"),
    ("pl",    "pl",      "Polish"),
    ("tr",    "tr",      "Turkish"),
    ("hi",    "hi",      "Hindi"),
    # Tier 3 — optional (broad global coverage)
    ("ja",    "ja",      "Japanese"),
    ("ko",    "ko",      "Korean"),
    ("ar",    "ar",      "Arabic"),
    ("ru",    "ru",      "Russian"),
    ("zh-CN", "zh-Hans", "Chinese (Simplified)"),
    ("uk",    "uk",      "Ukrainian"),
    ("cs",    "cs",      "Czech"),
    ("sv",    "sv",      "Swedish"),
    ("el",    "el",      "Greek"),
    ("ro",    "ro",      "Romanian"),
]

def parse_srt(content):
    blocks = re.split(r'\n{2,}', content.strip())
    parsed = []
    for block in blocks:
        lines = block.strip().split('\n')
        if len(lines) >= 3 and lines[0].strip().isdigit():
            parsed.append((lines[0].strip(), lines[1].strip(), ' '.join(lines[2:])))
    return parsed

def build_srt(blocks):
    return '\n\n'.join(f"{i}\n{ts}\n{txt}" for i, ts, txt in blocks) + '\n'

def translate_srt(srt_content, google_code):
    blocks  = parse_srt(srt_content)
    out     = []
    xlator  = GoogleTranslator(source='en', target=google_code)
    for idx, ts, txt in blocks:
        try:
            translated = xlator.translate(txt) or txt
        except Exception:
            translated = txt
        out.append((idx, ts, translated))
        time.sleep(0.05)
    return build_srt(out)

results = {}
errors  = []

for g_code, yt_code, name in LANGUAGES:
    try:
        results[yt_code] = {"name": name, "srt": translate_srt(SRT_CONTENT, g_code)}
        print(f"✅ {name} ({yt_code})")
    except Exception as e:
        errors.append(f"❌ {name}: {e}")
        print(f"❌ {name}: {e}")

print(f"\nDone: {len(results)}/25 languages")
if errors:
    print('\n'.join(errors))

print("\n[RESULTS_JSON_START]")
print(json.dumps({k: {"name": v["name"], "srt": v["srt"]} for k, v in results.items()},
    ensure_ascii=False))
print("[RESULTS_JSON_END]")
```

**Expected runtime:** ~2-3 minutes for all 25 languages.

---

## Step 2: Save All SRTs to Google Drive

For each language in results, save to the `Captions/` subfolder
(same folder as the English SRT, created by Agent 05).

**File naming convention:**
`[title-slug]_[yt-code]_sub_timestamps.srt`

Examples:
- `ghl-agency-setup_es_sub_timestamps.srt`
- `ghl-agency-setup_pt-BR_sub_timestamps.srt`
- `ghl-agency-setup_km_sub_timestamps.srt`

Use Drive MCP (tool prefix `mcp__4dfc969f-edaf-4938-b2b2-5a86f07b939f`) for each language:
```
create_file(
  title: "[title-slug]_[yt-code]_sub_timestamps.srt",
  textContent: "[translated SRT content]",
  contentMimeType: "text/plain",
  parentId: "[Captions subfolder ID]"
)
```
Save all 25 sequentially. Log each returned `webViewLink` for the summary report.

---

## Step 3: Upload to YouTube

### Primary Path — YouTube MCP
Try `mcp__youtube__update_video_metadata` first.
Check if it accepts a `captions` or `subtitles` parameter with language code + SRT content.
If yes: use it for each translated SRT file.

### Secondary Path — Python + YouTube Data API v3
If the YouTube MCP does not support caption upload, run this script:

```python
import subprocess, sys
subprocess.run([sys.executable, "-m", "pip", "install",
    "google-api-python-client", "google-auth-oauthlib",
    "--break-system-packages", "-q"], check=True)

from googleapiclient.discovery import build
from googleapiclient.http import MediaInMemoryUpload

VIDEO_ID = "PASTE_VIDEO_ID_HERE"
# creds = [load OAuth token — same credentials used by YouTube MCP]

def upload_caption(youtube, video_id, lang_code, lang_name, srt_content):
    body = {"snippet": {
        "videoId": video_id, "language": lang_code,
        "name": lang_name, "isDraft": False
    }}
    media = MediaInMemoryUpload(srt_content.encode('utf-8'),
        mimetype='application/octet-stream', resumable=False)
    return youtube.captions().insert(
        part="snippet", body=body, media_body=media).execute()
```

Note: requires OAuth credentials accessible from Python context.
If credentials are not available, proceed to fallback.

### Fallback — Drive Folder (Manual Upload)
**NEVER use Claude in Chrome for this — too token-expensive.**

If automated upload fails:
1. Share the Captions Drive folder link
2. Print this message:

```
⚠️ Automatikus YouTube upload nem sikerült — de minden fájl kész a Drive-on.

Manual upload (kb. 5 perc az összes nyelvhez):
YouTube Studio → Videók → kattints a videóra → "Feliratok" tab
→ minden nyelvnél: "ADD LANGUAGE" → Upload file → select SRT from Drive → Save

Drive mappa (25 SRT fájl): [CAPTIONS_FOLDER_LINK]
```

---

## Output Summary

After all steps complete, print to chat:

```
✅ Agent 05b — Multilingual Captions Complete
Video: [title] | ID: [video_id]

Translated & saved to Drive: [N]/25 → [Captions folder link]
YouTube upload: [✅ Automated | ⚠️ Manual — Drive folder shared above]

Tier 1 (10):  Spanish ✅ | Portuguese BR ✅ | German ✅ | French ✅ | Italian ✅
              Vietnamese ✅ | Thai ✅ | Khmer ✅ | Filipino ✅ | Malay ✅
Tier 2 (5):   Indonesian ✅ | Dutch ✅ | Polish ✅ | Turkish ✅ | Hindi ✅
Tier 3 (10):  Japanese ✅ | Korean ✅ | Arabic ✅ | Russian ✅ | Chinese ✅
              Ukrainian ✅ | Czech ✅ | Swedish ✅ | Greek ✅ | Romanian ✅

SEO impact: Video now indexed in 25 languages on YouTube search.
Cost: $0.00
```

---

## Technical Notes

- **Library:** `deep-translator` — uses Google Translate web interface unofficially. No API key.
  Free. Auto-installed via pip.
- **Rate limiting:** 50ms delay per subtitle block. 25 languages on a 10-min video ≈ 2-3 min.
- **Khmer (km):** Functional quality for SEO indexing; not literary.
- **RTL languages (Arabic, Hebrew):** YouTube handles right-to-left rendering natively from SRT.
- **Storage:** ~15-25 KB per SRT × 25 = ~500 KB per video. Negligible.
- **Backfill:** Works on any previously published video. Operator provides old video URL and
  English SRT (download from YouTube Studio → Subtitles if not in Drive).
- **Future:** When YouTube MCP adds `captions.insert`, Step 3 becomes fully automated.

---

## Post-Publish Menu Reference

Agent 05b is one of three optional post-publish steps (all run independently, in any order):

| Agent | What it does | Trigger phrase |
|-------|-------------|----------------|
| **05b** | Multilingual captions — 25 languages, free | "felirat fordítás" / "05b" |
| 03c | YouTube Shorts — 3-5 clips from long-form | "shortok" |
| 07  | Repurposing briefs — Reels, LinkedIn, email | "repurpose" |
