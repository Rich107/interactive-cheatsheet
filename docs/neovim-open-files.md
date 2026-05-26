# neovim — opening files & diffing revisions

Every useful way to open files in Neovim, including diffing files across git revisions. Covers shell-level `nvim` invocations and the in-nvim ex commands that are the natural counterpart for diff/split workflows.

Captured against `nvim` v0.11.6 on macOS. Ex command details from `:help`. The `:Gvdiffsplit` variants require the [vim-fugitive](https://github.com/tpope/vim-fugitive) plugin.

## 1. Open a single file

```bash
nvim src/app.ts
```

Jump straight to a specific line by passing `+LINE` (or the equivalent `-c "LINE"`):

```bash
nvim +42 src/app.ts
```

Jump to the first match of a pattern with `+/PATTERN`:

```bash
nvim +/TODO src/app.ts
```

## 2. Open multiple files at once

By default, extra files become buffers in the args list — `:next` / `:prev` walks them:

```bash
nvim src/app.ts src/utils.ts
```

Splits / tabs at startup:

```bash
nvim -o src/app.ts src/utils.ts   # horizontal splits
nvim -O src/app.ts src/utils.ts   # vertical splits
nvim -p src/app.ts src/utils.ts   # one tab per file
```

## 3. Read-only mode

```bash
nvim -R src/app.ts   # read-only (writes possible with :w!)
nvim -M src/app.ts   # modifications hard-disabled
view src/app.ts      # alias for nvim -R
```

## 4. Recover from a swap file

```bash
nvim -r                # list recoverable swap files
nvim -r src/app.ts     # recover the swap for this file
```

## 5. Run an ex command at startup

```bash
nvim -c "set number" src/app.ts
nvim -c "set number" -c "Gitsigns" src/app.ts   # multiple -c run in order
nvim --cmd "set rtp+=~/dev/myplugin" src/app.ts # runs BEFORE vimrc/init.lua
```

## 6. Pipe content into nvim

```bash
git log --oneline | nvim -        # read stdin into a scratch buffer
rg --color=never TODO | nvim -R - # stdin, read-only
```

## 7. Open files from a list

Load a quickfix file (e.g. compiler output, grep results):

```bash
nvim -q errors.qf
```

Open every file containing a pattern (ripgrep + shell expansion):

```bash
nvim $(rg -l "TODO")
```

## 8. Run nvim non-interactively (scripted)

```bash
nvim --headless -c ':help' -c ':q'                    # smoke-test help system
nvim --headless +'lua print(vim.version().major)' +qa # one-shot lua exec
nvim -es -c '%s/foo/bar/g' -c 'wq' src/app.ts         # silent ex-mode edit
```

## 9. Comparing files (diff mode)

From the shell:

```bash
nvim -d src/app.ts src/utils.ts
```

From inside nvim:

```vim
:diffsplit src/utils.ts   " open second file in a horizontal split, diff against current
:vert diffsplit src/utils.ts  " same, vertical
:diffthis                 " mark current window as a diff participant
:diffoff                  " turn diff off in current window
]c                        " next diff hunk (normal-mode keybinding)
[c                        " previous diff hunk
do                        " :diffget  — pull change from the OTHER window
dp                        " :diffput  — push change to the OTHER window
```

## 10. Diff a file against a git revision

With [vim-fugitive](https://github.com/tpope/vim-fugitive):

```vim
:Gvdiffsplit              " current file vs HEAD, side by side
:Gvdiffsplit main         " current file vs main branch
:Gvdiffsplit HEAD~1       " current file vs previous commit
:Gdiffsplit               " same idea, horizontal split
```

Plain vim alternative without fugitive — read `git show` output into a scratch buffer, then diff both windows:

```vim
:vert new | r !git show HEAD~1:./src/app.ts | 0d_ | setlocal buftype=nofile | diffthis
:wincmd p | diffthis
```

## 11. Use neovim as git's diff/merge tool

Configure once:

```bash
git config --global diff.tool nvimdiff
git config --global merge.tool nvimdiff
git config --global difftool.prompt false
```

Then:

```bash
git difftool main HEAD -- src/app.ts        # open the diff in nvim -d
git difftool HEAD~1 -- src/app.ts           # vs previous commit
git mergetool                               # walk every conflicted file
```

## 12. Open at the precise location of a git conflict

Jump to the first `<<<<<<<` marker on open — escape the angle brackets in the shell:

```bash
nvim +/'^<<<<<<< ' src/app.ts
```

Combine with `-O` to open every conflicted file side by side:

```bash
nvim -O +/'^<<<<<<< ' $(git diff --name-only --diff-filter=U)
```
