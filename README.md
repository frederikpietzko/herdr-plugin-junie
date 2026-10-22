# herdr-plugin-junie

A [Herdr](https://herdr.dev) plugin that adds [Junie CLI](https://junie.jetbrains.com) support. With it, Junie panes show up in Herdr like natively supported agents:

- Herdr shows Junie's `idle`, `working` and `blocked` state in the sidebar and in `herdr agent list`.
- Herdr notifies you when Junie finishes or waits for a permission decision.
- `herdr agent wait` and other automation that waits on agent state works with Junie.
- After a Herdr server restart, the pane reopens the same Junie session (`junie --session-id=<id> --resume`).

## Does this need native support in Herdr?

No. Herdr has a documented way for an agent to support Herdr without changes to Herdr ([Add Herdr support to your agent](https://herdr.dev/docs/add-herdr-support/)). The agent reports its state, resume command and exit through `herdr pane report-agent` / `pane release-agent`. Junie CLI has lifecycle hooks (`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `Stop`, `StopFailure`, `SessionEnd`). This plugin connects the Junie hooks to those Herdr reports.

Herdr's built-in screen detection, where Herdr reads the agent's screen without hooks, does need a Herdr release, because detection manifests can only patch agents that the Herdr binary already knows. This plugin does not need that.

## Requirements

- Herdr 0.9.2 or later (resume commands need 0.9.2)
- A Junie CLI version with hooks (EAP at the time of writing)
- `jq` (used by the installer to edit `~/.junie/config.json`)
- macOS or Linux

## Install

```sh
herdr plugin install frederikpietzko/herdr-plugin-junie
herdr plugin action invoke frederikpietzko.junie.install-hooks
```

For local development, link the working tree instead:

```sh
herdr plugin link /path/to/herdr-plugin-junie
herdr plugin enable frederikpietzko.junie
herdr plugin action invoke frederikpietzko.junie.install-hooks
```

Restart the Junie sessions that are already running. Junie reads hooks only when it starts.

The install action:

1. Copies `bin/junie-herdr-hook` to `~/.local/share/herdr-plugin-junie/` (or `$XDG_DATA_HOME/herdr-plugin-junie/`). GitHub-installed plugins move to a new directory on every update, so Junie cannot run the hook from the plugin directory. The plugin's startup hook copies the script again after an update.
2. Adds one hook entry per Junie event to `~/.junie/config.json`. Your other hooks stay. The installer writes a backup to `~/.junie/config.json.bak-herdr-junie` first.

Other actions:

```sh
herdr plugin action invoke frederikpietzko.junie.status
herdr plugin action invoke frederikpietzko.junie.uninstall-hooks
herdr plugin action invoke frederikpietzko.junie.install-skill
herdr plugin action invoke frederikpietzko.junie.uninstall-skill
```

You can also run `bin/junie-herdr-install install|uninstall|status|refresh|install-skill|uninstall-skill` directly. The skill actions require the `herdr` CLI on `PATH` (or `HERDR_BIN_PATH` pointing to it).

The Herdr skill action writes the output of `herdr --skill` to `~/.junie/skills/herdr/SKILL.md`, so Junie can learn how to use Herdr when relevant. The plugin refreshes this file after updates only if it was installed by this plugin; an existing unmanaged skill is left untouched. Set `JUNIE_SKILLS_DIR` to override the skills root directory.

## How it works

| Junie hook          | Report to Herdr                                   |
| ------------------- | ------------------------------------------------- |
| `SessionStart`      | `idle`, plus the session ID and resume command    |
| `UserPromptSubmit`  | `working`                                         |
| `PreToolUse`        | `working`                                         |
| `PermissionRequest` | `blocked`                                         |
| `Stop`              | `idle`                                            |
| `StopFailure`       | `idle`, with the error class as the message       |
| `SessionEnd`        | release the pane                                  |

Junie does not fire a hook in these cases:

- you interrupt a task with Esc
- a chat-mode task finishes (`Stop` does not run for chat tasks)
- you answer a permission dialog

For these cases, the hook starts one small watcher per pane. The watcher reads the pane's live screen once per second (`herdr pane read --source detection`) and corrects the state:

- a Junie spinner means `working`
- the permission dialog means `blocked`
- anything else means `idle`

A change counts only after two identical reads in a row. The watcher stops when the Junie process exits, and releases the pane if `SessionEnd` did not.

The resume command keeps these launch flags: `--brave`, `--sandbox`, `--model=`, `--effort=`, `--provider=`, `--config-location=`, `--agent-mode=`, `--model-location=`, `--skip-update-check` and `--use-local-cache`. Junie stores brave mode and the selected model in `~/.junie/settings.json` too. Batch runs (`junie "task"`, `--task`, `--review`, …) report their state but no resume command.

### Safety: `PermissionRequest` must be async

When a synchronous Junie `PermissionRequest` hook exits with status 0, Junie approves the action automatically. The installer therefore always adds this hook with `"async": true`. Junie then still shows the dialog, and the hook only reports `blocked`. Do not remove `async` from that entry.

## Configuration

Set these environment variables in the shell that starts Junie:

| Variable                   | Effect                                         |
| -------------------------- | ---------------------------------------------- |
| `JUNIE_HERDR=0`            | Turn off reporting for this Junie process      |
| `JUNIE_HERDR_POLL_SECONDS` | Watcher poll interval (default `1`)            |

Outside a Herdr pane (`HERDR_ENV` not set), the hook does nothing.

## Limitations

- After an interrupt, a finished chat task or an answered permission dialog, the state changes after about 2 seconds, because only the watcher sees these changes.
- The watcher recognizes Junie's screen by its spinner and the text of its permission dialog. A change to the Junie UI can break this until the patterns in `screen_state` are updated. The hook-driven transitions still work.
- Junie reads hooks only from `~/.junie/config.json`, `--config-location` files and installed extensions. It ignores project-level hooks.
- ACP mode (`--acp`, used by IDEs) does not run hooks, so the plugin cannot report on it.

## Test the integration

```sh
junie                       # in a Herdr pane
herdr agent list            # the pane shows agent "junie"
herdr pane get "$HERDR_PANE_ID"
```

To test session restore without touching your normal session, use `herdr --session junie-test`, start Junie, stop the session with `herdr session stop junie-test`, and start it again.

## License

MIT, see [LICENSE](LICENSE).
