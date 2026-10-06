  

# =============================  
NEOVIM SHORTCUT CHEAT SHEET

## MODES

i -> insert mode  
a -> insert after cursor  
o -> new line below  
crl + e -> return to normal mode

## NAVIGATION

h -> move left  
j -> move down  
k -> move up  
l -> move right  
b -> beginning of previous word  
e -> end of next work  
gg -> go to top of file  
G -> go to bottom of file  
0 -> start of line  
$ -> end of line  
w -> next word  
b -> previous word  
number + k -> Moves to that number up  
number + j -> Moves to that number down

## EDITING

dd -> delete line  
yy -> copy line  
p -> paste  
u -> undo  
Ctrl+r -> redo  
x -> delete character  
cw -> change word

## SEARCH

/word -> search for "word"  
n -> next result  
N -> previous result

## FILES

:w -> save file  
:q -> quit  
:wq -> save and quit  
:q! -> quit without saving  
:qa -> quit all files

## SPLITS (GENERIC VIM)

:split -> horizontal split  
:vsplit -> vertical split  
Ctrl+w h -> move to left split  
Ctrl+w j -> move to bottom split  
Ctrl+w k -> move to top split  
Ctrl+w l -> move to right split  
Ctrl+w c -> close split

## FILE EXPLORER (GENERIC)

:Ex  
:Explore

# =============================  
CUSTOM KEYBINDS FROM init.lua

Ctrl+n -> toggle file explorer (NvimTree)

Ctrl+h -> move to split left  
Ctrl+j -> move to split down  
Ctrl+k -> move to split up  
Ctrl+l -> move to split right

Ctrl+s -> save file  
Ctrl+q -> quit file

Ctrl+p -> file search (Telescope if installed)

# =============================  
NVIMTREE FILE EXPLORER KEYS

Enter -> open file  
a -> create file/folder  
d -> delete  
r -> rename  
c -> copy  
x -> cut  
p -> paste  
R -> refresh

# =============================  
THEMES

:colorscheme gruvbox  
:colorscheme dracula  
:colorscheme nord  
:colorscheme catppuccin  
:colorscheme tokyonight

# =============================  
PLUGINS

:PackerSync -> install/update plugins  
:NvimTreeToggle -> toggle file explorer