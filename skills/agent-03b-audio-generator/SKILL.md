---
name: agent-03b-audio-generator
description: Use this agent to generate ElevenLabs voiceover audio from an approved script. Triggers when the operator says "generate audio", "hang generálás", "audio", "generate the voiceover", "mehet a hang", "ElevenLabs audio", or "TTS" after Agent 03 script approval. Reads the confirmed script from Google Drive, extracts clean spoken text, generates MP3 via ElevenLabs API using Zoltan's cloned voice, saves to local YouTube folder, and presents for operator approval before HeyGen lipsync.
---

# Agent 03b — ElevenLabs Audio Generator
**Phase:** Pre-Publishing (runs after Agent 03 script approved, before HeyGen video)
**Position in Flywheel:** Agent 03 Script → **[03b Audio]** → Agent 04 Thumbnail → Agent 05 Publish

---

## 🎙️ VOICE CONFIGURATION — CHANGE THIS TO SWAP VOICE

```
ACTIVE VOICE:     Zoltan BD 03 instant
VOICE ID:         6cr2WtfrqWUvTgXGIdDB
API KEY FILE:     C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env
MODEL:            eleven_turbo_v2_5
WORKSPACE:        Máté's Workspace (Grant plan — 33M credits)
```

**To swap voice in the future:**
1. Go to elevenlabs.io → switch to Máté's Workspace → My Voices
2. Click the new voice → copy Voice ID from the URL or info panel
3. Update VOICE ID and ACTIVE VOICE name above in this SKILL.md
4. The API key stays the same (it's workspace-level, not voice-level)

---

## Pre-Flight: Read Script from Drive

Use Drive MCP tools directly — do not ask the operator to paste the script:

1. **Read `_Active_Channel.gdoc`:** `search_files(query: "title = '_Active_Channel'")` → first result `id` → `read_file_content(fileId)` → parse line 1: `[Channel Name] | [Channel Folder ID]`. If missing → invoke `channel-manager`.
2. **Find 03_Scripts subfolder:** `search_files(query: "title = '03_Scripts' and parentId = '[channelFolderID]'")` → get subfolder ID
3. **Read most recent script:** `search_files(query: "parentId = '[03_Scripts_ID]'")` → sort by createdTime descending → `read_file_content(fileId)` of the most recent file

If no script found: "No approved script in 03_Scripts. Please run Agent 03 first."

---

## Step 1: Extract Clean Spoken Text

Extract ONLY the spoken narration from the raw script.

**REMOVE — these are not spoken:**
- Section headers in brackets: `[HOOK]`, `[OUTCOME]`, `[TESTIMONIAL]`, `[SUBSCRIBE CTA]`, `[STEP N: Title]`, `[ENGAGEMENT CTA]`, `[CONVERSION CTA]`, `[END SCREEN — 20 seconds]`
- Everything from `[THUMBNAIL CAPTURE NOTE]` onward
- B-roll directions: any line starting with `[B-ROLL:` or `[BROLL:`
- Recording directions: any line starting with `[RECORDING:`
- Gesture notes in parentheses: e.g., `(gesture left)`, `(hold up hand)`

**KEEP — concatenate into one clean block:**
- All spoken narration lines between the section headers
- Separate paragraphs with a single space

**Text safety rules (ElevenLabs reads punctuation literally):**
- Replace colons (:) with commas
- Replace em dashes (—) with commas
- Replace double hyphens (--) with commas
- Replace "I am" with "I'm"
- If text exceeds 2500 characters → see Step 3 (Long Script Handling)

---

## Step 2: Generate Audio + Alignment Data via ElevenLabs API

Use `mcp__Desktop_Commander__start_process` with this PowerShell command.
Fill in `$text` with the clean spoken text from Step 1.
Fill in `$outFile` with the correct filename (naming convention below).

This uses the `/with-timestamps` endpoint — it returns the MP3 **and** character-level timing
data in one call, which Step 2b uses to generate the SRT subtitle file automatically.

```powershell
try {
    # Read API key from local env file — never hardcode here
    $apiKey    = (Get-Content 'C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env' | Where-Object { $_ -match '^ELEVENLABS_API_KEY=' }) -replace '^ELEVENLABS_API_KEY=', ''
    $voiceId   = '6cr2WtfrqWUvTgXGIdDB'
    $text      = '[CLEAN SPOKEN TEXT FROM STEP 1]'
    $outFile   = 'C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\[FILENAME].mp3'
    $alignFile = $outFile -replace '\.mp3$', '_align.json'

    New-Item -ItemType Directory -Force -Path 'C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio' | Out-Null

    $bodyObj = @{
        text     = $text
        model_id = 'eleven_turbo_v2_5'
        voice_settings = @{
            stability         = 0.5
            similarity_boost  = 0.75
            style             = 0.3
            use_speaker_boost = $true
        }
    }
    $body = [System.Text.Encoding]::UTF8.GetBytes((ConvertTo-Json $bodyObj -Depth 5))

    $headers = @{
        'xi-api-key'   = $apiKey
        'Content-Type' = 'application/json'
    }

    $response = Invoke-RestMethod `
        -Uri "https://api.elevenlabs.io/v1/text-to-speech/$voiceId/with-timestamps" `
        -Method POST `
        -Headers $headers `
        -Body $body

    # Save MP3
    $audioBytes = [Convert]::FromBase64String($response.audio_base64)
    [System.IO.File]::WriteAllBytes($outFile, $audioBytes)
    Write-Host "SUCCESS - $($audioBytes.Length) bytes"
    Write-Host "FILE: $outFile"

    # Save alignment JSON for SRT generation (Step 2b)
    $response.alignment | ConvertTo-Json -Depth 5 | Set-Content -Path $alignFile -Encoding UTF8
    Write-Host "ALIGN: $alignFile"
} catch {
    Write-Host "ERROR: $_"
}
```

**File naming convention:**
`[YYYY-MM-DD]_Audio_[title-slug]_BD03.mp3`
Example: `2026-05-16_Audio_ai-agents-content-workflow_BD03.mp3`

After running, use `mcp__Desktop_Commander__read_process_output` to confirm both
`SUCCESS` and `ALIGN:` lines appear in the output.

---

## Step 2b: Generate SRT Subtitle File from Alignment Data

Run immediately after Step 2. Calls the reusable Python script `generate_srt.py`
which reads the `_align.json`, groups characters into word-level subtitle blocks,
and writes a `.srt` file alongside the MP3.

**Script location:** `C:\Users\gdp\Documents\Claude\Projects\YouTube\Scripts\generate_srt.py`

Fill in `$alignFile` / `$srtFile` with the same base filename used in Step 2.
Set `$offset = 0` for a single-part script.
For multi-part scripts, set `$offset` to the **Last seconds** value from the previous part's result (see Step 3).

```powershell
$py        = "C:\Python314\python.exe"
$script    = "C:\Users\gdp\Documents\Claude\Projects\YouTube\Scripts\generate_srt.py"
$alignFile = "C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\[FILENAME]_align.json"
$srtFile   = "C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\[FILENAME].srt"
$offset    = 0   # float seconds — 0 for single-part; "Last seconds" from previous part for multi-part

& $py $script $alignFile $srtFile $offset
Start-Sleep 1

# Read result file (Python writes here because stdout isn't captured by Desktop Commander)
$result = $srtFile + ".result"
if (Test-Path $result) { Get-Content $result } else { Write-Host "ERROR: no result file — check Python path" }
```

**Result line format on success:**
`SUCCESS | Blocks: 42 | Last timestamp: 00:08:14,300 | Last seconds: 494.300 | SRT: C:\...\file.srt`

After running, confirm `SUCCESS` in the result. Note **Last seconds** — use as `$offset` for the next part.

---

## Step 3: Long Script Handling (>2500 characters)

If the clean spoken text exceeds 2500 characters:
- Count characters first: `$text.Length` in PowerShell
- Split at sentence boundaries (". " — period + space)
- Run **Step 2 + Step 2b** for each part in sequence:
  - Part 1: `_part01.mp3` + `_part01_align.json` → `_part01.srt` (timeOffset = 0)
  - Part 2: `_part02.mp3` + `_part02_align.json` → `_part02.srt` (timeOffset = Last timestamp from Part 1)
  - Part N: same pattern, cumulative offset
- After all parts are done, combine the SRT files into one:
  - Renumber subtitle entries sequentially across all parts
  - Save as `[FILENAME]_combined.srt`
- Inform operator: "Script was long — generated N parts. Combine the MP3s in Audacity before uploading to HeyGen. Combined SRT is ready for YouTube."

Typical YouTube scripts (8-10 min) run 4,000-6,000 characters — expect 2-3 parts.

---

## Step 4: Present for Operator Approval

After Steps 2 + 2b succeed, confirm to operator using computer:// links for both files:

```
✅ Audio + Subtitles generated!

Voice: Zoltan BD 03 instant | Model: eleven_turbo_v2_5
[▶️ Meghallgat](computer://C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\FILENAME.mp3)
[📄 SRT megnéz](computer://C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\FILENAME.srt)

🎧 Hallgasd meg. Válasz:
- "jó" / "approved" / "mehet" → tovább a HeyGen lipsync-re
- "újra" + visszajelzés → újragenerálás módosítással
- "hangot cserélek" → adj meg új Voice ID-t, újragenerálás
```

---

## Step 5: On Approval — HeyGen Lipsync Handoff

When operator approves, respond:

```
✅ Audio jóváhagyva!

MP3: [computer://C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\FILENAME.mp3]
SRT: [computer://C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\FILENAME.srt]

Következő lépés: Agent 04 (Thumbnail — Canva + Nano Banana 2 brief)

Az MP3-at az Agent 05 végén, a HeyGen Production Prompt generálásakor töltöd fel HeyGen-be.
Az SRT fájlt az Agent 05 YouTube feltöltésnél használja — automatikusan feltölti
a YouTube-ra a publishing folyamat részeként.

Workflow emlékeztető: 03b Audio ✅ → 04 Thumbnail → 05 Publish + HeyGen prompt (utolsó)
```

**HeyGen Avatar ID (default):** `bfcee432248442539191df03f922f575`

---

## Error Handling

| Hiba | Teendő |
|------|--------|
| 401 Unauthorized | API key lejárt vagy rossz workspace. Ellenőrizd a `.config\elevenlabs.env` fájlt |
| 404 Voice not found | A hang nincs Máté's Workspace-ben. Klónozd újra ott |
| Network error | Próbáld újra. Ha nem megy, ellenőrizd a VPN/internet kapcsolatot |
| File write error | Ellenőrizd hogy létezik-e a `YouTube\Audio\` mappa |
| Output too long | Darabold fel (lásd Step 3) |
| 0 bytes output | A szöveg üres volt — ellenőrizd a Step 1 szövegkivonást |
| SRT "ERROR: no result file" | Python path rossz — ellenőrizd hogy `C:\Python314\python.exe` létezik |
| SRT "ERROR: Alignment file not found" | Az `_align.json` hiányzik — futtasd újra a Step 2-t |
| SRT "ERROR: alignment JSON has no characters" | Az alignment JSON üres — Step 2 API hívás sikertelen volt |
| SRT "ERROR: No subtitle blocks generated" | Alignment adat hibás — ellenőrizd az `_align.json` tartalmát |

---

## Output Summary

| Mit | Hol |
|-----|-----|
| MP3 audio fájl | `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\` |
| SRT subtitle fájl | `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\` |
| Alignment JSON (temp) | `C:\Users\gdp\Documents\Claude\Projects\YouTube\Audio\` (törölhető a feltöltés után) |
| Config / API key | `C:\Users\gdp\Documents\Claude\Projects\YouTube\.config\elevenlabs.env` |
| Drive mentés | Opcionális — operator manuálisan feltöltheti |
