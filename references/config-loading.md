# Config Loading (Step 0 for every agent)

Every agent runs this before any other step. Orchestrators run it once and pass the
values on in the CONTEXT BLOCK; agents that get a CONTEXT BLOCK do not load it again.

## 1. Find the config

Use the first file that exists:

1. the path in the environment variable `YT_FLYWHEEL_CONFIG`
2. `~/.yt-flywheel/config.yaml` (`~` is the home folder of the current machine)
3. `config/config.yaml` in the plugin folder

If none exists: stop. Tell the user to copy `config/config.example.yaml` to
`~/.yt-flywheel/config.yaml` and fill it in. Do not guess values.

## 2. Use the values

In the skill files, `{key.path}` means a value from the config, for example
`{creator.name}`, `{paths.local_root}`, `{channel.youtube_id}`.
`{channel.*}` means the active channel's entry under `channels.list`.

If a key is empty or missing:
- `brand.guide_path` empty: use the generic rules in the skill.
- `integrations.funnel_skill` empty: skip funnel steps and say "no funnel skill set".
- `playbook.doc_id` empty: search Drive for `{playbook.title}`.
- Any other required key: stop and name the missing key.

## 3. Active channel

1. `_Channel_Pin.txt` in the connected working folder: the session is bound to that channel.
2. `@<key>` or `@<name>` in the message: match it against `channels.list` for this run.
3. `_Active_Channel.gdoc` in the Drive master folder `{drive.master_folder_name}`.
4. `channels.default`.
5. Nothing found: invoke `channel-manager`.

## 4. Find tools by name, never by id

Connector tool names carry a random id that differs per machine and per account
(`mcp__<id>__<tool>`). Never write or call a fixed id. Pick the tool by the end of its name:

| Need | Pick the tool whose name |
|---|---|
| Google Drive | ends with `__search_files`, `__read_file_content`, `__create_file`, `__update_file`, `__download_file_content` |
| vidIQ | contains `vidiq_` (for example `vidiq_channel_videos`); none found: use the estimate fallback silently |
| Canva | has `canva` in its server or tool name; optional |
| YouTube | `mcp__youtube__*` |
| Pipeline artifact | `mcp__cowork__create_artifact` / `update_artifact`; missing: show the status as text |
| Files and shell | `mcp__Desktop_Commander__*`; missing: use the built-in file and shell tools |
| Browser | a Chrome or built-in browser tool with the same action name (`navigate`, `file_upload`, `get_page_text`) |

If a needed tool is not found, try the call path once more, then tell the user which tool
is missing. Do not report a tool as missing from a startup warning alone.

## 5. Commands and aliases

Flow commands are in English (see `conventions.md`). If the user's message matches a key in
`triggers.aliases`, treat it as the English command it maps to.
