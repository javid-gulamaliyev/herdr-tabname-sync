# herdr-tabname-sync

A [herdr](https://herdr.dev) plugin that renames the Claude Code or Codex
session running in a tab whenever you rename the tab. Name the tab after what
the session is for, and `claude --resume <name>`, Codex thread resume and the
agent's terminal title all carry the same name.

One-way only: the herdr tab label is the source of truth. Renaming the session
from inside the agent does not rename the tab.

## Install

```sh
herdr plugin install javid-gulamaliyev/herdr-tabname-sync
herdr server reload-config
```

Requires herdr 0.9.0 or newer and `jq` on `PATH`.

## What it does

On `tab.renamed`, the plugin looks up the tab's new label and, for every pane
in that tab running a supported agent, sends the agent its rename command:

| Agent       | Command            |
| ----------- | ------------------ |
| Claude Code | `/rename <label>`  |
| Codex       | `/rename <label>`  |

The command is typed into the agent's composer and submitted with
`herdr agent prompt`, so it only goes out when that is safe:

- the agent is `idle` or `done`, not `working` or `blocked`;
- the composer is empty (nothing after the last prompt glyph on screen except
  the agent's own placeholder text), so the command is never appended to text
  you are typing;
- the agent is not on a permission prompt (`herdr agent prompt` refuses with
  `agent_blocked` before sending anything).

If any of those fail, the rename is queued and retried on the pane's next
`pane.agent_status_changed` event. A pane whose terminal title already equals
the label is left alone.

## Adding an agent

Agents are matched on the pane's `agent` field from `herdr api snapshot`. To
support another one, add a `case` arm to `rename_command` in
`bin/tabname-sync` that prints the agent's rename command, and add its `agent`
value to the pane filter in `tab_renamed`. If the agent shows placeholder text
in an empty composer, add that text to `composer_is_empty`.

## State

Queued renames live in the plugin's state directory
(`~/.local/state/herdr/plugins/javid-gulamaliyev.tabname-sync` by default):

```
pending/<pane_id>    desired session name for a rename not yet delivered
```

Deleting a pending file cancels that rename.

## Troubleshooting

```sh
herdr plugin log list --plugin javid-gulamaliyev.tabname-sync
```

shows every hook run with its stderr. Lines starting with `tabname-sync:` say
why a rename was deferred or dropped.

A status change can fire before the agent has redrawn its screen, so the
composer check retries for a few seconds before deferring. If a rename still
shows as deferred after the agent has gone idle, it is delivered on the pane's
next status change, which for an idle session is the end of its next turn.

Codex sets its terminal title to `<name> | <directory>`; the plugin treats that
as already synced. If Codex changes its empty-composer placeholder text, the
check will defer renames until `composer_is_empty` learns the new text.

## Development

```sh
git clone https://github.com/javid-gulamaliyev/herdr-tabname-sync ~/herdr-tabname-sync
herdr plugin link ~/herdr-tabname-sync
herdr server reload-config
```

## License

MIT
