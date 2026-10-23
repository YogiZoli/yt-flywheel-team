# Config

Every agent in this plugin reads one config file before it starts. The file holds
everything that belongs to you: your name, your channels, your folders, your brand guide
and your own trigger phrases. The plugin itself holds no personal values.

## Setup

1. Copy `config.example.yaml` to `~/.yt-flywheel/config.yaml`.
2. Fill in your values. Leave a value empty if you do not use it.
3. Add one entry under `channels.list` per YouTube channel.

The agents look for the config in this order and use the first one found:

1. the path in the environment variable `YT_FLYWHEEL_CONFIG`
2. `~/.yt-flywheel/config.yaml`
3. `config/config.yaml` inside this repo

Option 3 is ignored by git. Never commit a real config.

## Channels

Name a channel in a message with `@<key>` or `@<name>` (for example `new video @main`)
to use it for one run. Without that, the agents use the channel pin or the active channel,
see `references/config-loading.md`.

## Your own trigger phrases

Skill triggers are in English. To use phrases in another language, put them in
`triggers.aliases`. Inside a running flow the agents accept them as the English command.
To start a flow with them, add a small personal skill in `~/.claude/skills/` that lists the
phrases and points to the right agent. Keep it outside this repo.
