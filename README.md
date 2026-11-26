## herdr sessionizer
its a script that does everything awesome at all times

A [herdr](https://herdr.dev) port of tmux-sessionizer.
tmux concepts map to herdr like this:

| tmux    | herdr                        |
|---------|------------------------------|
| session | workspace (matched by label) |
| window  | tab                          |
| pane    | pane                         |

## Requirements
fzf, herdr and jq

On Linux, also socat or python3 (for socket API requests the herdr CLI has no command for).

## Usage
```bash
herdr-sessionizer [<partial name of session>]
```

if you execute herdr-sessionizer without any parameters it will FZF set of default directories or ones specified in config file.

The workspace label is the directory name as is (no `.` → `_` replacement like tmux).
If no herdr server is running, one is started.

### Switching clients
herdr applies focus changes to every attached client, so herdr-sessionizer moves only
the client that ran it like this:

- **outside herdr**: attach a client (run `herdr-sessionizer` in a terminal to start herdr).
- **inside herdr**: restart the calling client at the selected workspace. This only works for
  clients started by herdr-sessionizer. For any other client, all attached clients move.

To find the calling client, macOS compares terminal input times. Linux hides those
(the kernel updates them only every few seconds), so the clients' read byte counters
are compared before and after the fzf selection instead. On Linux, `-s` and a path
argument (nothing is typed while the script runs) move all attached clients.

### Hydrate files
When a new workspace is created, `<dir>/.herdr-sessionizer` (or else `~/.herdr-sessionizer`)
is sourced in the workspace's first pane, like tmux-sessionizer's `.tmux-sessionizer`.
It runs in the pane's shell, asynchronously, so herdr-sessionizer waits (up to 1 second)
until the file runs `herdr-sessionizer --ready [tab_id]`, then switches to `tab_id`
(or to the first tab). Call it before any blocking command such as `vim .`.
Create tabs with `--no-focus`: a focus change here would move every attached client.

```bash
# file: ~/.herdr-sessionizer
tab=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label scratch --no-focus |
    jq -r '.result.tab.tab_id // empty')
herdr-sessionizer --ready "$tab"   # show the scratch tab; plain --ready shows the first tab
vim .
```

## Session Commands
Session commands are for you to write / navigate without using herdr navigation commands.
They are meant to be used with zsh/vim remaps.

The basic idea is that you want long running commands on a per session basis.
You can start them by calling herdr-sessionizer with the -s option.  This will
start the long running session in the tab labeled 69.  This means if you open 6
tabs, it wont interfere with your way of using herdr.

### Example
herdr-sessionizer config file
```bash
# file: ~/.config/herdr-sessionizer/herdr-sessionizer.conf
TS_SESSION_COMMANDS=(opencode .)
```

There is one command which means you can call `herdr-sessionizer -s 0` only (`-s 1` is out of bounds)
This will effectively call the following commands:
```bash
herdr tab create --workspace $CURRENT_WORKSPACE --cwd $PWD --label 69
herdr pane run $NEW_PANE " exec opencode ."
```

### How i use it
Here are my vim remaps for herdr-sessionizer.  C-f will do the standard
sessionizer experience but Alt+h will mimic my harpoon navigation.  C-h is
first file harpoon.  M-h is first sessionizer command.

**vim**

fzf needs a terminal (`:!` has none), and creating a herdr tab would show up on every
attached client, so the script runs in an nvim terminal tab that closes when it exits.
```lua
local function sessionizer(args)
    return function()
        vim.cmd("tab terminal herdr-sessionizer " .. (args or ""))
        vim.cmd("autocmd TermClose <buffer> ++once bdelete!")
        vim.cmd("startinsert")
    end
end
vim.keymap.set("n", "<C-f>", sessionizer())
vim.keymap.set("n", "<M-h>", sessionizer("-s 0"))
vim.keymap.set("n", "<M-t>", sessionizer("-s 1"))
vim.keymap.set("n", "<M-n>", sessionizer("-s 2"))
vim.keymap.set("n", "<M-s>", sessionizer("-s 3"))
```

**zsh**
```bash
bindkey -s ^f "herdr-sessionizer\n"
bindkey -s '\eh' "herdr-sessionizer -s 0\n"
bindkey -s '\et' "herdr-sessionizer -s 1\n"
bindkey -s '\en' "herdr-sessionizer -s 2\n"
bindkey -s '\es' "herdr-sessionizer -s 3\n"
```

There is no herdr keybinding equivalent of the tmux `bind-key ... neww` remaps:
a pane or tab opened by a herdr keybinding shows up on every attached client.

## Enable Logs
This is for debugging purposes.

```bash
# file: ~/.config/herdr-sessionizer/herdr-sessionizer.conf
TS_LOG=file | echo # echo will echo to stdout, file will write to TS_LOG_FILE
TS_LOG_FILE=<file> # will write logs to <file> Defaults to ~/.local/share/herdr-sessionizer/herdr-sessionizer.logs
```
