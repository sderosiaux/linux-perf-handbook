# Modern CLI Replacements

Rust/Go tools replacing classic Unix utilities. Faster, better defaults, human-friendly output.

## File Operations

### eza (ls replacement)
```bash
eza -la --icons --git          # Long list + git status + icons
eza -T --level=2               # Tree view, 2 levels
eza -l --sort=modified         # Sort by mtime
eza --git-ignore               # Respect .gitignore
```
**Install:** `apt install eza` / `brew install eza` / `cargo install eza`

### bat (cat replacement)
```bash
bat file.py                    # Syntax highlighted
bat -p file.py                 # Plain (no line numbers)
bat -A file.txt                # Show non-printable chars
bat --diff file.py             # Git diff markers
bat -l json < data             # Force language
```
**Install:** `apt install bat` (binary: `batcat`) / `brew install bat`

### fd (find replacement)
```bash
fd pattern                     # Find files matching pattern
fd -e py                       # By extension
fd -H pattern                  # Include hidden
fd -I pattern                  # Don't respect .gitignore
fd pattern --exec rm {}        # Execute on matches
fd -t d pattern                # Directories only
fd -t f -x wc -l               # Count lines in all files
fd '\.rs$' --exec-batch wc -l  # Batch exec (faster)
```
**Install:** `apt install fd-find` (binary: `fdfind`) / `cargo install fd-find`

### ripgrep (grep replacement)
```bash
rg pattern                     # Recursive search
rg -i pattern                  # Case insensitive
rg -w pattern                  # Whole word
rg -l pattern                  # Files only
rg -C 3 pattern                # 3 lines context
rg -t py pattern               # Python files only
rg --hidden pattern            # Include hidden
rg -F 'literal'                # Fixed string (no regex)
rg -g '*.py' pattern           # Glob filter
rg --json pattern | jq         # JSON output
rg -U 'foo\nbar'               # Multiline
```
**Install:** `apt install ripgrep` / `cargo install ripgrep`

## Disk Usage

### dust (du replacement)
```bash
dust                           # Current directory
dust -n 10                     # Top 10 entries
dust -d 2                      # Max depth 2
dust -r                        # Reverse order
dust -b                        # No percent bars
```
**Install:** `apt install dust` / `cargo install du-dust`

### duf (df replacement)
```bash
duf                            # All filesystems
duf /home                      # Specific path
duf --json                     # JSON output
duf --only local               # Local only
duf --hide-mp "/snap/*"        # Hide mounts
```
**Install:** `apt install duf` / `brew install duf`

### ncdu (interactive du)
```bash
ncdu /                         # Scan root
ncdu -x /                      # Stay on one filesystem
ncdu -e --exclude .git         # Exclude patterns
```
**Install:** `apt install ncdu`

### gdu (fast ncdu alternative)
```bash
gdu                            # Current dir
gdu -d                         # Show apparent size
gdu -np                        # No cross-device, progress
```
**Install:** `apt install gdu`

## Process & System

### procs (ps replacement)
```bash
procs                          # All processes
procs --tree                   # Tree view
procs -w pattern               # Watch + filter
procs --sortd cpu              # Sort by CPU desc
procs java                     # Filter by name
```
**Install:** `cargo install procs`

### btop (top replacement)
```bash
btop                           # Launch dashboard
btop -lc                       # Low color mode
btop --utf-force               # Force UTF-8
```
**Install:** `apt install btop`

### bottom (btm)
```bash
btm                            # Launch
btm -b                         # Basic mode
btm --process_command          # Full command
```
**Install:** `cargo install bottom`

### glances
```bash
glances                        # Terminal mode
glances -w                     # Web server (:61208)
glances --export json          # JSON export
glances -C config.conf         # Custom config
glances --enable-plugin docker # Enable plugins
```
**Install:** `pip install glances` / `apt install glances`

## Network

### dog (dig replacement)
```bash
dog example.com                # A record
dog example.com MX             # MX records
dog example.com --json         # JSON output
dog @1.1.1.1 example.com       # Specific DNS
dog example.com --https        # DoH
dog example.com --tls          # DoT
```
**Install:** `brew install dog`

### doggo (alternative to dog)
```bash
doggo example.com              # A record
doggo example.com MX           # MX records
doggo @https://1.1.1.1/dns-query example.com  # DoH
doggo @quic://dns.adguard.com example.com     # DoQ
```
**Install:** `brew install doggo`

### gping (ping with graph)
```bash
gping google.com               # Single host
gping google.com cloudflare.com  # Multiple hosts
gping -4 example.com           # IPv4 only
gping --cmd "curl -s url"      # Graph command time
```
**Install:** `apt install gping` / `cargo install gping`

### xh (httpie-compatible, faster)
```bash
xh GET api.example.com         # GET
xh POST api.example.com name=John  # POST JSON
xh -d url                      # Download
xh --curl api.example.com      # Show curl equivalent
xh -b api.example.com          # Body only
xh -h api.example.com          # Headers only
```
**Install:** `cargo install xh`

### httpie
```bash
http GET api.example.com/users
http POST api.example.com name=John age:=30  # := for non-strings
http -v example.com            # Verbose
http --session=s1 example.com  # Named session
http -f POST example.com field=value  # Form
```
**Install:** `pip install httpie`

## Text Processing

### sd (sed replacement)
```bash
sd 'old' 'new' file.txt        # Replace in file
sd -F 'literal' 'new' file.txt # Fixed string
echo 'hello' | sd 'e' 'a'      # Pipe
sd 'foo(.*)bar' 'baz$1qux' f   # Capture groups
sd -s 'old' 'new' file.txt     # In-place
```
**Install:** `cargo install sd`

### choose (cut/awk replacement)
```bash
echo "a b c d" | choose 0      # First field
echo "a b c d" | choose -1     # Last field
echo "a b c d" | choose 1:3    # Range
echo "a:b:c" | choose -f ':' 0 # Custom delimiter
```
**Install:** `cargo install choose`

## Git & Navigation

### delta (git diff pager)
```gitconfig
# ~/.gitconfig
[core]
    pager = delta
[interactive]
    diffFilter = delta --color-only
[delta]
    navigate = true
    side-by-side = true
    line-numbers = true
[merge]
    conflictStyle = zdiff3
```
**Install:** `apt install git-delta` / `cargo install git-delta`

### zoxide (cd replacement)
```bash
z foo                          # Jump to best match
zi foo                         # Interactive with fzf
z -                            # Previous dir
zoxide query foo               # Preview match

# Setup (add to shell rc)
eval "$(zoxide init bash)"     # or zsh/fish
```
**Install:** `apt install zoxide` / `cargo install zoxide`

### fzf (fuzzy finder)
```bash
fzf                            # File picker
cat file | fzf                 # Filter input
fzf --preview 'bat {}'         # Preview with bat
fzf -m                         # Multi-select

# Shell integration
eval "$(fzf --bash)"           # Ctrl+R, Ctrl+T, Alt+C
```
**Install:** `apt install fzf`

## Bulk Install

```bash
# Ubuntu/Debian
sudo apt install ripgrep fd-find bat eza fzf btop git-delta zoxide duf gping ncdu

# Aliases for renamed binaries
alias fd='fdfind'
alias bat='batcat'
```

## Reference Table

| Classic | Modern | Lang | Key Feature |
|---------|--------|------|-------------|
| ls | eza | Rust | Git status, icons |
| cat | bat | Rust | Syntax highlighting |
| find | fd | Rust | Simple syntax, fast |
| grep | ripgrep | Rust | 10x faster |
| du | dust | Rust | Visual bars |
| df | duf | Go | Clean tables |
| ps | procs | Rust | Colors, tree |
| top | btop | C++ | Dashboard |
| dig | dog/doggo | Rust/Go | DoH/DoT |
| ping | gping | Rust | Graph |
| curl | xh | Rust | HTTPie syntax |
| sed | sd | Rust | Sane regex |
| cut | choose | Rust | Simple syntax |
| cd | zoxide | Rust | Frecency |
