---
name: pipeline-orchestrator
description: DEPRECATED — replaced by two new orchestrators. Use "video-production-orchestrator" for single video pipelines, or "strategic-launch-orchestrator" for playlist/funnel launch. Do NOT trigger this skill directly.
---

# ⚠️ DEPRECATED — Pipeline Orchestrator

This skill has been replaced by two purpose-built orchestrators:

## Use instead:

**For a new video (single video pipeline):**
→ Say: "új videó" or "start new video"
→ Triggers: **video-production-orchestrator**
→ Same 6-gate pipeline (01 → 02 → 03 → 03b → 04 → 05)
→ PLUS: automatically reads and injects the opt-in URL into all publishing artifacts

**For a new playlist / channel launch (funnel setup before first video):**
→ Say: "playlist launch" or "indítsuk el a playlistet"
→ Triggers: **strategic-launch-orchestrator**
→ Builds: playlist concept → lead magnet → landing page → GHL automation → opt-in URL

## Why the change?

The old pipeline-orchestrator did not distinguish between channel-level strategy
(what is the playlist? what is the lead magnet?) and video-level production
(script, audio, thumbnail, publish). The new two-orchestrator architecture separates these
cleanly, enabling parallel tracks: funnel can be built while the first video is in production.

## Migration

No data migration needed. All Drive outputs, folder structures, and agent skills are identical.
The only change is how you start a session:
- Old: "pipeline" / "run the pipeline"
- New video: "új videó" / "start new video"
- New launch: "playlist launch" / "funnel setup"
