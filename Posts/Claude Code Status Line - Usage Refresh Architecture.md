---
title: Claude Code Custom Status Line
tags:
  - claude-code
  - statusline
  - infrastructure
created: 2026-04-03
stage: evergreen
origin: ai-assisted
---

# Claude Code Custom Status Line

A custom status line for Claude Code showing a gradient context bar and git state. Inspired by [danielmiessler/Personal_AI_Infrastructure](https://github.com/danielmiessler/Personal_AI_Infrastructure). Tailwind-inspired colors shift green → yellow → orange → red as context fills. Responsive across four terminal width modes.

## Settings

Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "${HOME}/.claude/statusline.sh"
  },
  "contextDisplay": {
    "compactionThreshold": 83
  }
}
```

`compactionThreshold` controls when autocompaction triggers. The context bar rescales so it fills to 100% at the threshold rather than at literal context exhaustion.

## Script

Save to `~/.claude/statusline.sh` and `chmod +x`. Requires `jq` and `git`.

```bash
#!/bin/bash
set -o pipefail

# ── Parse input (Claude Code pipes JSON to stdin) ────────────────────────────

input=$(cat)
eval "$(echo "$input" | jq -r '
  "current_dir=" + (.workspace.current_dir // .cwd // "." | @sh) + "\n" +
  "context_pct=" + (.context_window.used_percentage // 0 | tostring)
' 2>/dev/null)"
context_pct=${context_pct:-0}

# ── Git info ─────────────────────────────────────────────────────────────────

is_git_repo=false
if git rev-parse --git-dir > /dev/null 2>&1; then
    is_git_repo=true
    branch=$(git branch --show-current 2>/dev/null)
    [ -z "$branch" ] && branch="detached"
    stash_count=$(git stash list 2>/dev/null | wc -l | tr -d ' ')
    [ -z "$stash_count" ] && stash_count=0
    sync_info=$(git rev-list --left-right --count HEAD...@{u} 2>/dev/null)
    last_commit_epoch=$(git log -1 --format='%ct' 2>/dev/null)
    if [ -n "$sync_info" ]; then
        ahead=$(echo "$sync_info" | awk '{print $1}')
        behind=$(echo "$sync_info" | awk '{print $2}')
    else
        ahead=0; behind=0
    fi
    [ -z "$ahead" ] && ahead=0
    [ -z "$behind" ] && behind=0
fi
dir_name=$(basename "$current_dir" 2>/dev/null || echo ".")

# ── Terminal width (3-tier: Kitty IPC → stty → tput) ────────────────────────

_width_cache="/tmp/claude-sl-width-${KITTY_WINDOW_ID:-default}"
detect_terminal_width() {
    local width=""
    if [ -n "$KITTY_WINDOW_ID" ] && command -v kitten >/dev/null 2>&1; then
        width=$(kitten @ ls 2>/dev/null | jq -r --argjson wid "$KITTY_WINDOW_ID" \
            '.[].tabs[].windows[] | select(.id == $wid) | .columns' 2>/dev/null)
    fi
    [ -z "$width" ] || [ "$width" = "0" ] || [ "$width" = "null" ] && \
        width=$(stty size </dev/tty 2>/dev/null | awk '{print $2}')
    [ -z "$width" ] || [ "$width" = "0" ] && width=$(tput cols 2>/dev/null)
    if [ -n "$width" ] && [ "$width" != "0" ] && [ "$width" -gt 0 ] 2>/dev/null; then
        echo "$width" > "$_width_cache" 2>/dev/null; echo "$width"; return
    fi
    if [ -f "$_width_cache" ]; then
        local cached; cached=$(cat "$_width_cache" 2>/dev/null)
        [ "$cached" -gt 0 ] 2>/dev/null && { echo "$cached"; return; }
    fi
    echo "${COLUMNS:-80}"
}
term_width=$(detect_terminal_width)
if   [ "$term_width" -lt 35 ]; then MODE="nano"
elif [ "$term_width" -lt 55 ]; then MODE="micro"
elif [ "$term_width" -lt 80 ]; then MODE="mini"
else MODE="normal"; fi

# ── Colors (Tailwind-inspired) ───────────────────────────────────────────────

RESET='\033[0m'
SLATE_600='\033[38;2;71;85;105m'
EMERALD='\033[38;2;74;222;128m'
ROSE='\033[38;2;251;113;133m'
CTX_PRIMARY='\033[38;2;129;140;248m'
CTX_SECONDARY='\033[38;2;165;180;252m'
CTX_BUCKET_EMPTY='\033[38;2;75;82;95m'
GIT_PRIMARY='\033[38;2;56;189;248m'
GIT_VALUE='\033[38;2;186;230;253m'
GIT_DIR='\033[38;2;147;197;253m'
GIT_CLEAN='\033[38;2;125;211;252m'
GIT_STASH='\033[38;2;165;180;252m'
GIT_AGE_FRESH='\033[38;2;125;211;252m'
GIT_AGE_RECENT='\033[38;2;96;165;250m'
GIT_AGE_STALE='\033[38;2;59;130;246m'
GIT_AGE_OLD='\033[38;2;99;102;241m'

# ── Helpers ──────────────────────────────────────────────────────────────────

# Gradient: Green(74,222,128) → Yellow(250,204,21) → Orange(251,146,60) → Red(239,68,68)
get_bucket_color() {
    local pos=$1 max=$2 pct=$((pos * 100 / max)) r g b
    if [ "$pct" -le 33 ]; then
        r=$((74 + (250 - 74) * pct / 33)); g=$((222 + (204 - 222) * pct / 33)); b=$((128 + (21 - 128) * pct / 33))
    elif [ "$pct" -le 66 ]; then
        local t=$((pct - 33))
        r=$((250 + (251 - 250) * t / 33)); g=$((204 + (146 - 204) * t / 33)); b=$((21 + (60 - 21) * t / 33))
    else
        local t=$((pct - 66))
        r=$((251 + (239 - 251) * t / 34)); g=$((146 + (68 - 146) * t / 34)); b=$((60 + (68 - 60) * t / 34))
    fi
    printf '\033[38;2;%d;%d;%dm' "$r" "$g" "$b"
}

render_context_bar() {
    local width=$1 pct=$2 output="" filled=$((pct * width / 100))
    [ "$filled" -lt 0 ] && filled=0
    local use_spacing=false; [ "$width" -le 20 ] && use_spacing=true
    for i in $(seq 1 $width 2>/dev/null); do
        if [ "$i" -le "$filled" ]; then
            output="${output}$(get_bucket_color $i $width)⛁${RESET}"
        else
            output="${output}${CTX_BUCKET_EMPTY}⛁${RESET}"
        fi
        [ "$use_spacing" = true ] && output="${output} "
    done
    echo "${output% }"
}

calc_bar_width() {
    local mode=$1 content_width=72 prefix_len suffix_len bucket_size
    case "$mode" in
        nano|micro) prefix_len=2;  suffix_len=5; bucket_size=2 ;;
        mini)       prefix_len=12; suffix_len=5; bucket_size=2 ;;
        normal)     prefix_len=12; suffix_len=5; bucket_size=1 ;;
    esac
    local buckets=$(((content_width - prefix_len - suffix_len) / bucket_size))
    case "$mode" in
        nano)   [ "$buckets" -lt 5 ]  && buckets=5  ;;
        micro)  [ "$buckets" -lt 6 ]  && buckets=6  ;;
        mini)   [ "$buckets" -lt 8 ]  && buckets=8  ;;
        normal) [ "$buckets" -lt 16 ] && buckets=16 ;;
    esac
    echo "$buckets"
}

# ── Output ───────────────────────────────────────────────────────────────────

# Context bar
SETTINGS_FILE="${HOME}/.claude/settings.json"
COMPACTION_THRESHOLD=$(jq -r '.contextDisplay.compactionThreshold // 100' "$SETTINGS_FILE" 2>/dev/null)
COMPACTION_THRESHOLD="${COMPACTION_THRESHOLD:-100}"
raw_pct="${context_pct%%.*}"; [ -z "$raw_pct" ] && raw_pct=0
if [ "$COMPACTION_THRESHOLD" -lt 100 ] && [ "$COMPACTION_THRESHOLD" -gt 0 ]; then
    display_pct=$((raw_pct * 100 / COMPACTION_THRESHOLD))
    [ "$display_pct" -gt 100 ] && display_pct=100
else display_pct="$raw_pct"; fi

if   [ "$display_pct" -ge 80 ]; then pct_color="$ROSE"
elif [ "$display_pct" -ge 60 ]; then pct_color='\033[38;2;251;146;60m'
elif [ "$display_pct" -ge 40 ]; then pct_color='\033[38;2;251;191;36m'
else pct_color="$EMERALD"; fi

bar_width=$(calc_bar_width "$MODE")
printf "${SLATE_600}────────────────────────────────────────────────────────────────────────${RESET}\n"
case "$MODE" in
    nano|micro) printf "${CTX_PRIMARY}◉${RESET} $(render_context_bar $bar_width $display_pct) ${pct_color}${raw_pct}%%${RESET}\n" ;;
    mini|normal) printf "${CTX_PRIMARY}◉${RESET} ${CTX_SECONDARY}CONTEXT:${RESET} $(render_context_bar $bar_width $display_pct) ${pct_color}${raw_pct}%%${RESET}\n" ;;
esac
printf "${SLATE_600}────────────────────────────────────────────────────────────────────────${RESET}\n"

# PWD & Git
if [ "$is_git_repo" = "true" ] && [ -n "$last_commit_epoch" ]; then
    age_s=$(($(date +%s) - last_commit_epoch))
    if   [ $((age_s / 60)) -lt 1 ];    then age_display="now";               age_color="$GIT_AGE_FRESH"
    elif [ $((age_s / 3600)) -lt 1 ];   then age_display="$((age_s / 60))m"; age_color="$GIT_AGE_FRESH"
    elif [ $((age_s / 3600)) -lt 24 ];  then age_display="$((age_s / 3600))h"; age_color="$GIT_AGE_RECENT"
    elif [ $((age_s / 86400)) -lt 7 ];  then age_display="$((age_s / 86400))d"; age_color="$GIT_AGE_STALE"
    else age_display="$((age_s / 86400))d"; age_color="$GIT_AGE_OLD"; fi
fi

case "$MODE" in
    nano)
        printf "${GIT_PRIMARY}◈${RESET} ${GIT_DIR}${dir_name}${RESET}"
        [ "$is_git_repo" = true ] && printf " ${GIT_VALUE}${branch}${RESET}"
        printf "\n" ;;
    micro)
        printf "${GIT_PRIMARY}◈${RESET} ${GIT_DIR}${dir_name}${RESET}"
        if [ "$is_git_repo" = true ]; then
            printf " ${GIT_VALUE}${branch}${RESET}"
            [ -n "$age_display" ] && printf " ${age_color}${age_display}${RESET}"
        fi; printf "\n" ;;
    mini)
        printf "${GIT_PRIMARY}◈${RESET} ${GIT_DIR}${dir_name}${RESET}"
        if [ "$is_git_repo" = true ]; then
            printf " ${SLATE_600}│${RESET} ${GIT_VALUE}${branch}${RESET}"
            [ -n "$age_display" ] && printf " ${SLATE_600}│${RESET} ${age_color}${age_display}${RESET}"
        fi; printf "\n" ;;
    normal)
        printf "${GIT_PRIMARY}◈${RESET} ${GIT_PRIMARY}PWD:${RESET} ${GIT_DIR}${dir_name}${RESET}"
        if [ "$is_git_repo" = true ]; then
            printf " ${SLATE_600}│${RESET} ${GIT_PRIMARY}Branch:${RESET} ${GIT_VALUE}${branch}${RESET}"
            [ -n "$age_display" ] && printf " ${SLATE_600}│${RESET} ${GIT_PRIMARY}Age:${RESET} ${age_color}${age_display}${RESET}"
            [ "$stash_count" -gt 0 ] && printf " ${SLATE_600}│${RESET} ${GIT_PRIMARY}Stash:${RESET} ${GIT_STASH}${stash_count}${RESET}"
            if [ "$ahead" -gt 0 ] || [ "$behind" -gt 0 ]; then
                printf " ${SLATE_600}│${RESET} ${GIT_PRIMARY}Sync:${RESET} "
                [ "$ahead" -gt 0 ] && printf "${GIT_CLEAN}↑${ahead}${RESET}"
                [ "$behind" -gt 0 ] && printf "${GIT_STASH}↓${behind}${RESET}"
            fi
        fi; printf "\n" ;;
esac
```

## Known Limitations

- Context percentage won't match `/context` exactly due to hook JSON excluding some internal overhead (~22k preloaded tokens)
- When Claude Code hits rate limits, its own status message can replace custom status line output

## History

Previously included Anthropic API usage tracking (5-hour/7-day rate limit windows), session cost estimates, and a `SessionEnd` hook for cache refresh. Removed 2026-04-03 due to reliability issues with the OAuth API integration. The usage data accounted for ~60% of the script complexity and all external dependencies (`python3`, `curl`, Keychain access).
