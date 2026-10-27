---
name: strategic-launch-orchestrator
description: Pointer skill (retired as an orchestrator). A playlist launch is TWO separate jobs with two separate owners, and this plugin does not chain them. Traffic side = playlist-strategy-orchestrator (this plugin). Funnel side = the funnel skill set in the config (T2). Triggers on the old launch phrases ("playlist launch", "channel launch", "new playlist", "playlist setup", "launch the playlist", "full launch", "before the first video") and routes the operator to the right one.
---

**Step 0. Load config:** follow `references/config-loading.md` (skip if a CONTEXT BLOCK is passed in). `{...}` values below come from that config.

# Strategic Launch — Pointer (not an orchestrator any more)

A "playlist launch" is two independent jobs. They are NOT chained here — coupling them was the
old design and it is gone. Route the operator:

## Job 1 — TRAFFIC: the playlist itself
**Skill:** `playlist-strategy-orchestrator` (this plugin, Layer A).
Trigger: `"playlist strategy"` / `"plan the playlist"`.
Produces: `PlaylistStrategy_[Series]` doc, one Video Brief Card per planned video, and
`playlist_brief.json` (the T1 → T2 handoff). Then it stops.

## Job 2 — FUNNEL: landing page + lead magnet + email sequences
**Owner:** the funnel skill set in `{integrations.funnel_skill}` (team T2), which also
hands off to the `campaign-ai-agent-team` plugin (T3) for the email sequences.
Trigger: `"start"` / `"lead magnet"` / `"funnel setup"`.
T2 reads `playlist_brief.json` on its own, builds the funnel on whatever platform it chooses
(GHL today), and writes `optin_registry.json` back — which `video-production-orchestrator` then
reads automatically for every video description. No manual coordination, no cross-plugin call
from here.

## The contract between them
See `references/layer-interface.md` §7 (T1 / T2 / T3 handoff files).

```
playlist-strategy-orchestrator ──playlist_brief.json──► CMO orchestrator (T2)
video-production-orchestrator   ◄──optin_registry.json── CMO orchestrator (T2)
```

## If the operator explicitly wants "everything, in order"
Tell them: run `"playlist strategy"` first, then `"start"` for the funnel. Two messages, two
jobs, each resumable on its own. That is the whole launch.
