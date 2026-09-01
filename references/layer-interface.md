# Layer Interface — Playlist Strategy ↔ Per-Video Execution

Single source of truth for the boundary between the two layers of this plugin.
Every skill that touches strategy or a video brief reads this file.

---

## 0. THE TWO LAYERS

```
┌─────────────────────────────────────────┐        ┌─────────────────────────────────────────┐
│  LAYER A — PLAYLIST STRATEGY             │        │  LAYER B — PER-VIDEO EXECUTION           │
│  runs once per playlist (+ re-plans)     │        │  runs once per video                     │
│                                         │        │                                         │
│  skills/strategy/                        │        │  skills/execution/                       │
│    playlist-strategy-orchestrator        │        │    video-production-orchestrator         │
│    agent-01-channel-strategist           │        │    agent-03-script-writer                │
│  skills/shared/                          │        │    agent-04-thumbnail                    │
│    agent-02-seo-title (Playlist Keyword  │        │    agent-05-launch-publish               │
│                        Mode)             │        │    agent-05b / agent-06 / agent-07 / 03c │
│                                         │        │  skills/shared/                          │
│                                         │        │    agent-02-seo-title (Execution Mode)   │
│                                         │        │                                         │
│  DECIDES:                               │        │  DECIDES:                               │
│  - playlist concept + positioning       │        │  - title wording (3 variants)           │
│  - umbrella keyword                     │  ───►  │  - description prose, chapters          │
│  - the ordered video sequence           │ Brief  │  - script content                      │
│  - per-video: topic, primary keyword,   │  Card  │  - thumbnail composition                │
│    search intent, angle, video_type,    │        │  - production format + publish copy     │
│    transformation, CTA lead magnet      │        │                                         │
│                                         │        │  NEVER decides keyword, angle, topic,   │
│  WRITES the PlaylistStrategy doc + one   │        │  video_type, or which lead magnet.      │
│  Video Brief Card per planned video.     │        │  READS them from the Brief Card.        │
└─────────────────────────────────────────┘        └─────────────────────────────────────────┘
                    ▲                                                   │
                    │              agent-06 analytics review            │
                    └───────────────────────────────────────────────────┘
                         feedback → Layer A re-plan (never into a running pipeline)
```

**One rule above all:** Layer B runs entirely inside the frame Layer A set. Layer B does NOT
decide `topic` / `angle` / `video_type` / `cta_lead_magnet` — those come from the Brief Card and
Layer B never changes them.

**Keyword is the one exception.** The Brief Card's `primary_keyword` is a broad playlist-level
anchor. Layer B (Agent 02 Execution Mode) always runs its own tighter, video-specific keyword
pass on top of it — the umbrella keyword is wide, the video keyword is narrow. This is deliberate
duplicate work. If the card has no keyword, Layer B researches from scratch and continues — it
never stops and never routes back to Layer A for a keyword.

---

## 1. THE INTERFACE ARTIFACT — VIDEO BRIEF CARD

The Video Brief Card is the single unit that crosses the boundary. Layer A produces one
per planned video; Layer B consumes exactly one per run.

**Location:** `01_Strategy/` subfolder of the active channel.
**Filename:** `Brief_[SeriesSlug]_EP[NN]_[topic-slug]` (e.g. `Brief_youtube-ownership_EP04_email-list`).
**Format:** plain-text Google Doc, `key: value` lines, one blank line between blocks.

### Schema (v1)

```
=== VIDEO BRIEF CARD v1 ===
brief_id:              [series-slug]-ep[NN]            # stable, unique within the channel
playlist:              [Series Name]
umbrella_keyword:      [playlist-level keyword]
episode:               [N] of [total]
arc:                   Opening | Middle | Closing
status:                planned | in_production | published

topic:                 [one sentence — what this video is about]
primary_keyword:       [broad playlist-level anchor keyword — Layer B refines this per video; may be blank]
secondary_keywords:    [comma-separated, optional]
search_intent:         informational | commercial | transactional
angle:                 [the specific take / framing — 1 sentence]
video_type:            talking_head | heygen_avatar | screen_recording
sr_audio_mode:         live | elevenlabs_vo | n/a       # only when video_type = screen_recording

transformation_promise:[what the viewer can do after watching]
cta_lead_magnet:       [lead magnet name + slot code, e.g. "Ownership Playbook PDF (LM1a)"]
cta_type:              email_optin | next_video | related_video
next_episode_topic:    [topic of EP N+1, for the "Next Video" CTA — or "series finale"]

authority_hook:        [known name/tool to borrow search volume, or "none"]
notes:                 [free text for the execution layer — optional]
=== END VIDEO BRIEF CARD v1 ===
```

### Who writes which field

| Field | Written by | May Layer B change it? |
|---|---|---|
| `topic`, `angle`, `video_type`, `cta_lead_magnet`, `playlist`, `episode`, `arc` | Layer A only | **No** |
| `primary_keyword` / `secondary_keywords` | Layer A sets the anchor | Layer B **refines** per video (Agent 02 §B0) — records the tighter video keyword, keeps the anchor as secondary |
| `status` | Layer A sets `planned` | Layer B advances `planned → in_production → published` |

If Layer B believes `topic` / `angle` / `video_type` / `cta_lead_magnet` is wrong, it STOPS and
reports: `"A Brief Card [brief_id] [mező] mezője nem stimmel. Ez stratégiai döntés — futtasd a
playlist-strategy-orchestrator-t az újratervezéshez."` It does not edit those and continue.
A missing or weak `primary_keyword` is NOT a reason to stop — Layer B just researches it.

---

## 2. THE PLAYLIST STRATEGY DOC

The umbrella document. One per playlist. The Brief Cards are its rows made concrete.

**Filename:** `PlaylistStrategy_[SeriesName]` in `01_Strategy/`.

### Schema (v1)

```
=== PLAYLIST STRATEGY v1 ===
series_name:           [Series Name]
series_slug:           [series-slug]
created:               [YYYY-MM-DD]
last_replanned:        [YYYY-MM-DD or "never"]
status:                active | paused | complete

audience:              [who this playlist catches — from the Avatar Card]
positioning:           [how this playlist is positioned vs the channel's other playlists]
umbrella_keyword:      [the playlist-level keyword all videos ladder up to]
transformation_arc:    [the single transformation the whole playlist delivers]

lead_magnet:           [name + slot code]
optin_url:             [URL or "pending — set by strategic-launch-orchestrator"]
cta_language:          [the consistent CTA phrasing used across every video in the series]

default_video_type:    talking_head | heygen_avatar | screen_recording
target_type_mix:       Evergreen [X]% / Viral [Y]% / Depth [Z]%   # guide: 65 / 25 / 10

--- VIDEO SEQUENCE ---
EP01 | [topic] | keyword: [kw] | type: [Evergreen/Viral/Depth] | video_type: [..] | brief: [emitted/pending]
EP02 | ...
...
EP12 | ...
--- END SEQUENCE ---

keyword_map_check:     [1 line confirming no two EPs target the same primary_keyword]
=== END PLAYLIST STRATEGY v1 ===
```

---

## 3. RESOLUTION — WHICH BRIEF CARD DOES A VIDEO RUN USE

Layer B (`video-production-orchestrator`) resolves the Brief Card at Step 0, first hit wins:

1. **Explicit in the trigger** — `"új videó EP04"` or `"új videó brief: youtube-ownership-ep04"`.
2. **Next `planned` card in sequence** — read `PlaylistStrategy_[Series]`, take the lowest-EP
   card whose `status: planned`. Show it, confirm at Gate 1.
3. **No PlaylistStrategy doc and no card** → tell the operator:
   `"Nincs playlist stratégia ehhez a csatornához. Futtasd: 'playlist stratégia'. Vagy egyszeri videó playlist nélkül: 'ad-hoc videó'."`
4. **`"ad-hoc videó"`** → the ONE sanctioned escape hatch (see §4).

---

## 4. AD-HOC SINGLE VIDEO (escape hatch)

A single video with no playlist is allowed. It is the exception, not the path.

- Trigger: `"ad-hoc videó"` / `"egyszeri videó"` / `"ad-hoc video"`.
- Layer B generates a **minimal Brief Card inline** by asking the operator 2 things: topic and
  video_type. `primary_keyword` is left blank — Agent 02 §B0 researches it. All other fields:
  `arc: n/a`, `playlist: AD-HOC`, `cta_lead_magnet` = the lead magnet from any `optin_registry.json`
  entry for the channel if present, else `email_optin` generic.
- The card is saved to `01_Strategy/` as `Brief_ADHOC_[date]_[slug]` with `status: in_production`.
- Individual Layer B agents (`agent-02` … `agent-05`) also run fully standalone with no
  orchestrator and no card — they fall back to their own Pre-Flight Drive reads as written
  in each SKILL.md. "Playlist nélkül, pipeline nélkül" is supported.

---

## 5. FEEDBACK LOOP

`agent-06-analytics-reviewer` (Layer B, post-publish) writes a review to `06_Analytics_Reviews/`
with a **CONTENT DIRECTION UPDATE** section. That section is an input to Layer A, not to any
running pipeline. Acting on it means:

- Operator says `"playlist újratervezés"` → `playlist-strategy-orchestrator` re-runs in
  re-plan mode: reads the latest analytics review, reorders / rewrites the `planned` Brief
  Cards from the current EP onward, bumps `last_replanned`. Already-`published` cards are
  never touched.

A breakthrough video flagged by agent-06 does **not** auto-change anything. It waits for the
operator to trigger a re-plan.

---

## 6. CONTEXT BLOCK CARRIES THE HANDOFF

When an orchestrator invokes an agent it passes the CONTEXT BLOCK (conventions.md §1).
As of v2 the block carries `BRIEF_ID` and the playlist fields are sourced from the Brief
Card, not re-derived. Agents receiving `BRIEF_ID` treat the named card as read-only truth.

---

## 7. CROSS-TEAM CONTRACT (T1 ↔ T2 ↔ T3)

This plugin (`yt-flywheel-team`) is **T1** in the freedomOS 3-team system. The Video Brief Card
above is INTERNAL to T1 (per video, within a playlist). The team boundaries use different files.
Canonical: `github.com/YogiZoli/freedomOS` → `docs/architecture.md`.

| File | Writer | Reader | Contains | This plugin |
|---|---|---|---|---|
| `playlist_brief.json` | **T1** | T2 | `playlist_id`, `playlist_topic`, `primary_keyword`, `avatar_pain_point`, `script_summary`, `video_count` | `playlist-strategy-orchestrator` writes it at S3c, then stops |
| `optin_registry.json` | T2 | **T1** | `playlist_id`, `opt_in_url`, `lead_magnet_type`, `lead_magnet_title`, `entry_tag`, `platform`, `platform_meta{}`, `confirmed_date` | `video-production-orchestrator` reads **only `opt_in_url`** at Step 0b (per `playlist_id`) |
| `campaign_brief.json` | T2 | T3 | `entry_tag`, `entry_workflow_id`, segment tags, `platform` | not touched here |

Rules:
- **T1 never invokes a T2 skill.** It writes `playlist_brief.json` and stops. T2 (the CMO
  orchestrator in `email-list-building`) picks it up on its own.
- **Platform-agnostic:** T1 reads only the neutral top-level fields (`opt_in_url`, and for display
  `lead_magnet_title`). `platform` (default `gohighlevel`) names T2's tool; `platform_meta` holds
  tool-specific IDs and is T2/T3 only. Swapping GHL for another tool changes nothing in T1.
- If `optin_registry.json` is missing when a video is produced, T1 uses a placeholder in the
  description and does not block — the funnel can come later and the operator can re-run publish.
- `strategic-launch-orchestrator` is a pointer only. It chains nothing.
