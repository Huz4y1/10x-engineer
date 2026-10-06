Everything here works in a bare `nvim` with no `init.lua`, no plugins, nothing. This is the fallback for a fresh machine, a server over SSH, or a broken config.

Nothing on this page needs anything installed. For your own binds see [[Keybinds]].

Getting out

The one thing to know before anything else.

```
  Esc          back to normal mode
  Ctrl+[       same thing, easier to reach

  :q           quit
  :q!          quit, throw away changes
  :wq  or  ZZ  save and quit
  :qa!         quit everything, no matter what
```

If you are ever lost, press `Esc` twice and you are in normal mode.

Modes

```
                    ┌──────────────┐
              i a o │              │ Esc
         ┌─────────►│    INSERT    │─────────┐
         │          └──────────────┘         │
         │                                   ▼
  ┌──────┴───────┐                    ┌─────────────┐
  │    NORMAL    │◄───────────────────│   NORMAL    │
  │ (the default)│                    └─────────────┘
  └──────┬───────┘
         │  v V Ctrl+v                  :
         ▼                              ▼
  ┌──────────────┐              ┌──────────────┐
  │    VISUAL    │              │  COMMAND :   │
  └──────────────┘              └──────────────┘
```

Normal mode is home. You spend most of your time there, not in insert.

The grammar

This is the thing that makes vim make sense. Commands are a sentence:

```
        [count]  operator  [count]  motion/text object
           2         d                  w          = delete 2 words
                     d                  $          = delete to end of line
           3         >                  j          = indent 3 lines down
                     y                  i(         = yank inside parentheses
                     c                  it         = change inside HTML tag
```

You do not memorise hundreds of commands. You learn ~10 operators and ~20 motions, and every combination works. That is the whole design.

Doubling an operator makes it act on the line: `dd` delete line, `yy` yank line, `>>` indent line.

Operators

```
  d      delete (cuts, goes into a register)
  c      change (delete then insert)
  y      yank (copy)
  >  <   indent / unindent
  =      auto-indent
  gu gU  lowercase / uppercase
  g~     swap case
  gq     reformat to textwidth
  !      filter through an external command
```

Motions

```
  CHARACTER      h j k l          left down up right
                 0                start of line (column 0)
                 ^                first non-blank character
                 $                end of line

  WORD           w  b             next / previous word start
                 e  ge            next / previous word end
                 W B E            same but whitespace-separated only
                                  (so "foo.bar" is one W, three w)

  FIND IN LINE   f<char>          jump forward onto that char
                 t<char>          jump forward up to before it
                 F<char> T<char>  same, backwards
                 ;  ,             repeat that find forward / backward

  LINES          gg               top of file
                 G                bottom of file
                 42G  or  :42     go to line 42
                 }  {             next / previous blank line
                 %                jump to matching bracket

  SCREEN         H M L            top / middle / bottom of screen
                 zz zt zb         centre / top / bottom the cursor line
                 Ctrl+d  Ctrl+u   half page down / up
                 Ctrl+f  Ctrl+b   full page forward / back
```

`f` and `t` are underused. `dt)` means "delete up to the closing paren" and is faster than any amount of `l`.

Text objects

Motions go from the cursor to somewhere. Text objects select a whole thing regardless of where the cursor sits inside it. Prefix with `i` (inner) or `a` (a, includes the delimiters).

```
  iw  aw      word
  is  as      sentence
  ip  ap      paragraph
  i"  a"      double-quoted string
  i'  a'      single-quoted
  i(  a(      parentheses      (also ib / ab)
  i[  a[      square brackets
  i{  a{      braces           (also iB / aB)
  i<  a<      angle brackets
  it  at      HTML/XML tag
```

```
  cursor anywhere inside these quotes:
      let msg = "hello there world";
                 ^
      ci"   ->  let msg = "";           cursor in insert mode
      ca"   ->  let msg = ;

  cursor anywhere inside the parens:
      foo(a, b, c)
            ^
      di(   ->  foo()
```

`ci"`, `ci(`, `cit` and `dap` will be most of your editing.

Entering insert mode

```
  i  I      insert before cursor / at first non-blank of line
  a  A      append after cursor / at end of line
  o  O      open new line below / above
  s  S      substitute character / whole line
  C         change to end of line          (same as c$)
  gi        jump to where you last left insert mode
```

Inside insert mode

```
  Ctrl+w    delete the word before the cursor
  Ctrl+u    delete to start of line
  Ctrl+r "  paste the unnamed register
  Ctrl+r +  paste the system clipboard
  Ctrl+o    run ONE normal mode command, then come back
  Ctrl+n    complete word from this buffer      <- no plugin needed
  Ctrl+p    same, searching backwards
```

`Ctrl+o zz` to centre the screen mid-typing is a nice one.

Visual mode

```
  v          character-wise
  V          line-wise
  Ctrl+v     BLOCK-wise (columns)
  gv         reselect whatever you had selected last
  o          jump to the other end of the selection
```

Block mode is the one worth practising:

```
  Ctrl+v, select down 4 lines, then:
     I  text  Esc      insert "text" at the start of ALL those lines
     A  text  Esc      append at the end of all of them
     d                 delete the rectangle
     $A ;  Esc         append ; to the end of every line, ragged or not
```

Undo and redo

```
  u          undo
  Ctrl+r     redo
  U          undo all recent changes on one line

  g-  g+     move backwards / forwards through the undo TREE
  :earlier 10m       state as of ten minutes ago
  :later 5m
```

Nvim keeps a tree, not a stack. If you undo and then type, the old branch still exists — `g-` reaches it. This has saved a lot of work.

Registers

Yanking and deleting goes into registers. `"x` before a command picks one.

```
  "ayy       yank line into register a
  "ap        paste from register a
  "Ayy       APPEND this line to register a (capital = append)

  ""         unnamed, the default
  "0         last yank only, NOT overwritten by deletes
  "1-"9      previous deletes, shifting down
  "_         black hole, deletes without clobbering anything
  "%         current filename
  ".         last inserted text

  :reg       show everything currently held
```

Two that matter in practice. `"0p` pastes your last *yank* after you have done some deletes that would otherwise have overwritten it. `"_d` deletes without touching your clipboard.

System clipboard

```
  "+y       yank to system clipboard
  "+p       paste from system clipboard
  "+yy      whole line to clipboard
  "+yG      from cursor to end of file to clipboard
```

If `"+y` does nothing, you have no clipboard provider. `:checkhealth` will say so. On Linux install `xclip` or `wl-clipboard`; on Windows and macOS it normally works out of the box.

Search

```
  /pattern       search forward
  ?pattern       search backward
  n  N           next / previous match
  *  #           search for the word under the cursor, forward / back

  :nohlsearch    clear the highlighting  (:noh is enough)
  Ctrl+L         also clears it, and this IS a default binding in Nvim
```

`*` is excellent — put the cursor on a variable name, press `*`, and cycle every use with `n`.

Search and replace

```
  :s/old/new/        first match on this line
  :s/old/new/g       every match on this line
  :%s/old/new/g      every match in the file
  :%s/old/new/gc     ask for confirmation each time
  :5,20s/old/new/g   only lines 5 to 20
  :'<,'>s/old/new/g  only the visual selection (typing :s in visual gives you this)
```

Flags: `g` all on the line, `c` confirm, `i` ignore case, `e` no error if not found.

Global commands

`:g` runs a command on every line matching a pattern. Very powerful, worth knowing two forms:

```
  :g/TODO/d              delete every line containing TODO
  :v/keep/d              delete every line NOT containing "keep"   (v = inverse)
  :g/^$/d                delete all blank lines
  :g/error/normal A;     append ; to every line containing "error"
```

Marks and jumps

```
  ma         set mark a at the cursor (lowercase = this file)
  'a         jump to the line of mark a
  `a         jump to the exact position of mark a
  mA         capital = global, works across files

  ``         jump back to where you were before the last jump
  ''         same, but line-wise
  Ctrl+o     go back through the jump list
  Ctrl+i     go forward again
  g;         go back through the CHANGE list (where you last edited)
  `.         jump to the last change
```

`Ctrl+o` is the "go back" button. Chase a definition across three files, then `Ctrl+o` a few times to get home.

Macros

Record a sequence, replay it a hundred times.

```
  qa          start recording into register a
  ...         do the edits
  q           stop recording
  @a          play it back
  @@          play the last macro again
  100@a       play it 100 times
  :%normal @a run it on every line
```

Build the macro so it ends positioned for the next iteration (usually with `j0`), then `100@a` and let it run off the end of the file — it stops on error.

Repeat

```
  .        repeat the last change
  ;  ,     repeat the last f/t/F/T motion forward / backward
  &        repeat the last :s on this line
```

The dot command is the most valuable key in vim. `cw` a word, `Esc`, then `n.` `n.` `n.` through a file is faster than writing a substitute for small jobs.

Files and buffers

Every open file is a buffer. Buffers are not windows.

```
  :e file        open a file
  :e!            reload from disk, discarding changes
  :w             write
  :w file        write to a different name
  :sav file      save as, and switch to the new file

  :ls  or  :buffers    list buffers
  :b 3                 go to buffer 3
  :b partialname       go to buffer by fuzzy name
  :bn  :bp             next / previous buffer
  :bd                  close the buffer
  Ctrl+^               toggle between the last two buffers
```

`Ctrl+^` is the fastest way to flip between a file and its test.

Windows and splits

```
  :sp   or  Ctrl+w s      horizontal split
  :vs   or  Ctrl+w v      vertical split
  :sp file                split and open a file

  Ctrl+w h j k l          move to the split in that direction
  Ctrl+w w                cycle
  Ctrl+w c   or  :q       close this split
  Ctrl+w o                close every OTHER split

  Ctrl+w H J K L          move the split itself to that edge
  Ctrl+w =                make all splits equal size
  Ctrl+w _                maximise height
  Ctrl+w |                maximise width
  10 Ctrl+w +             grow by 10 rows
```

Note `Ctrl+w h` is the vanilla way to move between splits. Your config remaps this to plain `Ctrl+h`, which will not exist on a fresh machine.

Tabs

Tabs in vim hold layouts of windows, they are not file tabs like VS Code.

```
  :tabnew        new tab
  :tabe file     open a file in a new tab
  gt  gT         next / previous tab
  3gt            go to tab 3
  :tabc          close tab
  :tabo          close all other tabs
```

Netrw, the built-in file explorer

There is a file browser built in. No NvimTree needed.

```
  :Ex        open it in the current window   (:Explore)
  :Sex       in a horizontal split
  :Vex       in a vertical split
  :Lex       as a left sidebar, toggles       <- closest to NvimTree
  -          go up one directory

  inside netrw:
  Enter      open file or enter directory
  %          create a new file
  d          create a new directory
  D          delete
  R          rename
  i          cycle the view (thin / long / wide / tree)
  gh         toggle hidden files
```

`:Lex` then `Ctrl+w l` to get back to your code is a fine no-plugin workflow.

Completion without plugins

Insert mode has real completion built in.

```
  Ctrl+n  Ctrl+p       words from open buffers
  Ctrl+x Ctrl+f        FILE PATHS - genuinely great
  Ctrl+x Ctrl+l        whole lines
  Ctrl+x Ctrl+o        omni completion (language aware, needs LSP or filetype plugin)
  Ctrl+x Ctrl+s        spelling suggestions

  in the popup:  Ctrl+n / Ctrl+p to move, Enter to accept, Ctrl+e to cancel
```

`Ctrl+x Ctrl+f` completing a path as you type an import is worth remembering.

Modern Nvim defaults

Recent Neovim ships bindings vanilla vim never had. Check `:version` if unsure.

```
  Nvim 0.10+
  gcc            comment / uncomment the line       <- built in now
  gc{motion}     comment a motion, e.g. gcap
  gc  (visual)   comment the selection

  Nvim 0.11+
  grn            LSP rename
  gra            LSP code action
  grr            LSP references
  gri            LSP implementation
  Ctrl+s         signature help (insert mode)
  K              hover documentation

  ]d  [d         next / previous diagnostic
  ]q  [q         next / previous quickfix item
  ]b  [b         next / previous buffer
```

The `gr*` ones only do something when a language server is actually attached.

Quickfix

A list of locations you step through. Populated by `:grep`, `:make`, or LSP.

```
  :grep pattern .    search the project (uses your system grep)
  :copen             open the quickfix window
  :cnext  :cprev     step through results  (or ]q / [q)
  :cfirst  :clast
  :cclose
```

Folding

```
  zf{motion}    create a fold
  za            toggle the fold under the cursor
  zo  zc        open / close it
  zR            open every fold
  zM            close every fold

  :set foldmethod=indent    fold by indentation, sane default
  :set foldmethod=syntax
```

Terminal mode

```
  :terminal        open a shell in a buffer
  :sp | terminal   in a split
  Ctrl+\ Ctrl+n    leave terminal mode back to normal mode
  i                back into terminal mode
```

That escape sequence is unguessable and the reason people think the built-in terminal is broken.

Help, the actual superpower

With no config and no internet, the docs are all local.

```
  :help              the front page
  :help dd           help for a normal-mode command
  :help i_CTRL-w     insert-mode mapping (i_ prefix)
  :help 'number'     an OPTION, note the quotes
  :help :split       an ex command, note the colon
  :help user-manual  a genuinely good book, start at usr_02

  Ctrl+]             follow the tag under the cursor
  Ctrl+o             go back
  :helpgrep pattern  search all help files
```

The prefixes matter: `i_` for insert mode, `v_` for visual, `c_` for command line, quotes for options, colon for commands.

Useful settings when you have no config

Type these into a bare nvim to make it liveable for a session.

```vim
:set number relativenumber   " line numbers, relative for easy 12j
:set ignorecase smartcase    " case-insensitive unless you type a capital
:set hlsearch incsearch      " highlight matches, jump as you type
:set expandtab               " spaces not tabs
:set tabstop=4 shiftwidth=4
:set mouse=a                 " mouse works, useful for resizing splits
:set clipboard=unnamedplus   " y and p use the system clipboard directly
:set wrap linebreak          " wrap on word boundaries
:syntax on
```

`:set clipboard=unnamedplus` is the big one — it makes plain `y` and `p` talk to the system clipboard so you stop typing `"+`.

Checking things

```
  :checkhealth      what is broken, missing providers, clipboard problems
  :version          version and compile flags
  :scriptnames      every file that was sourced (empty-ish = no config loaded)
  :map              every current mapping
  :verbose map <key>   shows WHERE a mapping was defined
  :echo stdpath('config')   where nvim expects your init.lua
```

If your config is not loading, `:scriptnames` and `:echo stdpath('config')` will tell you why in about five seconds.

Where the config goes

```
  Linux / macOS      ~/.config/nvim/init.lua
  Windows            ~/AppData/Local/nvim/init.lua
```

See [[Configuration]] for the actual setup, and [[Keybinds]] for the custom binds on top of all this.
