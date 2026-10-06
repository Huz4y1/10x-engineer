---
tags: [moc, nvim, editor, tools]
---

# Nvim

Section: [[Tools]]

---

## The three notes

**[[Default Nvim]]** — every **built-in** shortcut. Works in a bare `nvim` with no config, no plugins, nothing. The fallback for a fresh machine, a server over SSH, or a broken config.

**[[Configuration]]** — setting it up: `init.lua`, plugin manager, LSP, treesitter, and the Windows/WSL specifics.

**[[Keybinds]]** — your own custom binds, layered on top of the defaults.

---

## Which note do I want?

| Situation | Go to |
|---|---|
| "How do I delete a word?" | [[Default Nvim]] |
| "I'm SSH'd into a server with no config" | [[Default Nvim]] |
| "How do I install/set this up?" | [[Configuration]] |
| "What did *I* bind that to?" | [[Keybinds]] |
| "It's broken" | [[Configuration]] — troubleshooting |

## The 30-second version

```
Esc          normal mode (home)
i            insert before cursor
:w  :q  :wq  write, quit, both
dd  yy  p    delete line, copy line, paste
u   Ctrl+r   undo, redo
/text  n     search, next match
:help x      docs for x
```

> **If you are ever lost, press `Esc` twice.** You're in normal mode.

## Why bother

The point isn't speed — it's that **vim's grammar composes**. You learn ~10 operators and ~20 motions, and every *combination* works without being memorised separately. `d` + `iw` = delete inner word. `c` + `a"` = change everything inside quotes. `y` + `3j` = yank 3 lines down.

That's why [[Default Nvim]] teaches the grammar before the commands.

## Related

[[Tools]] · [[Dev environment - Git, Docker, CLI]] · [[03 — PROGRAMMING]]
