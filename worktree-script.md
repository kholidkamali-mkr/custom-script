# Git Worktree Helpers (`wtv`)

Shell functions for managing git worktrees. Source from `~/.config/shell/wtv.sh`.

## Features

- Create isolated worktrees per ticket
- Auto-copy personal/gitignored files (CLAUDE.md, prompts, mcp.json, settings.json)
- Sync VS Code color + ticket label into mcp.json (for HITL)
- Cleanup, listing, and stale detection

## Setup

```bash
# Add once to ~/.zshrc
for file in ~/.config/shell/*.sh; do
    [[ -f "$file" ]] && source "$file"
done

# Then save the script below to:
~/.config/shell/wtv.sh
```

## Workflow

```
1. Pick up ticket (e.g. MOB-1234-some-feature)
         |
         v
2. wtv MOB-1234-some-feature
   -> fetches latest master
   -> creates worktrees/MOB-1234-some-feature with new branch
   -> copies personal files (CLAUDE.md, mcp.json, settings.json, prompts)
   -> opens VS Code in isolated worktree
   -> extension auto-sets unique color on open
         |
         v
3. wtv-sync (run once in the worktree terminal)
   -> reads color from .vscode/settings.json (set by extension)
   -> extracts ticket label (MOB-1234)
   -> updates mcp.json headers: X-Workspace-Color + X-Workspace-Label
         |
         v
4. Open Claude Code in that VS Code terminal
   -> claude works in the worktree (clean, isolated)
   -> no interference with other tickets
         |
         v
5. Work the ticket (Phase 1-4 from CLAUDE.md)
   -> clarify -> plan -> implement -> test
   -> commit & push from the worktree
         |
         v
6. Create PR, get it reviewed & merged
         |
         v
7. wtv-rm MOB-1234-some-feature
   -> removes worktree (force, safe for gitignored files)
   -> optionally deletes local branch
   -> clean slate
         |
         v
8. Next ticket -> back to step 1
```

### Parallel Work

You can have multiple worktrees open at once. Working on MOB-1234 but need a hotfix?
Just `wtv MOB-5678` in another terminal. No stashing, no branch juggling.

## Quick Reference

```bash
wtv MOB-1234           # start ticket
wtv MOB-5678           # start another in parallel
wtv-sync               # sync color + label into mcp.json (run after VS Code opens)
wtv-ls                 # what's active?
wtv-rm MOB-1234        # done with ticket, clean up
wtv-stale              # find merged worktrees safe to remove
wtv-prune              # prune stale git references
```

## Full Script

```bash
# Git Worktree helpers

wtv() {
    local BRANCH=$1
    local BASE=${2:-master}
    local REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
    local DIR=$REPO_ROOT/worktrees/$BRANCH

    # Validate branch name provided
    if [[ -z "$BRANCH" ]]; then
        echo "Usage: wtv <branch-name> [base-branch]"
        return 1
    fi

    # Check we're inside a git repo
    if [[ -z "$REPO_ROOT" ]]; then
        echo "Error: not inside a git repository"
        return 1
    fi

    # Bail if worktree dir already exists
    if [[ -d "$DIR" ]]; then
        echo "Directory already exists: $DIR"
        return 1
    fi

    # Fetch latest from remote without switching branches
    echo "Fetching latest $BASE from origin..."
    git fetch origin "$BASE" || { echo "Error: git fetch failed"; return 1; }

    # Create worktree branched off the updated remote base
    echo "Creating worktree for branch '$BRANCH' in $DIR..."
    git worktree add -b "$BRANCH" "$DIR" "origin/$BASE" || { echo "Error: worktree creation failed"; return 1; }

    # Copy personal files (gitignored, not in worktree by default)
    echo "Copying personal files..."
    local files=(
        ".github/prompts/hitl.prompt.md"
        "CLAUDE.md"
        ".vscode/mcp.json"
        ".vscode/settings.json"
    )
    for file in "${files[@]}"; do
        if [[ -f "$REPO_ROOT/$file" ]]; then
            mkdir -p "$DIR/$(dirname "$file")"
            cp "$REPO_ROOT/$file" "$DIR/$file"
            echo "  copied $file"
        else
            echo "  skipped $file (not found)"
        fi
    done

    echo ""
    echo "Worktree ready! Opening VS Code..."
    code -n "$DIR"
    echo ""
    echo "After VS Code opens, run 'wtv-sync' to sync color + label into mcp.json"
}

# Sync VS Code color (from extension) + ticket label into mcp.json
wtv-sync() {
    local settings=".vscode/settings.json"
    local mcp=".vscode/mcp.json"
    local BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)

    if [[ ! -f "$settings" ]]; then
        echo "No .vscode/settings.json found"
        return 1
    fi

    if [[ ! -f "$mcp" ]]; then
        echo "No .vscode/mcp.json found"
        return 1
    fi

    # Read color from settings.json (set by VS Code extension)
    local color=$(grep -o '"activityBar.activeBackground": "[^"]*"' "$settings" | head -1 | cut -d'"' -f4)
    if [[ -z "$color" ]]; then
        echo "No activityBar.activeBackground found in $settings"
        echo "Make sure the VS Code color extension has set the color first."
        return 1
    fi

    # Extract ticket label (MOB-1234-blabla -> MOB-1234)
    local label=$(echo "$BRANCH" | grep -oE '^[A-Z]+-[0-9]+' || echo "$BRANCH")

    # Update mcp.json headers
    sed -i '' "s|\"X-Workspace-Color\": \"[^\"]*\"|\"X-Workspace-Color\": \"$color\"|" "$mcp"
    sed -i '' "s|\"X-Workspace-Label\": \"[^\"]*\"|\"X-Workspace-Label\": \"$label\"|" "$mcp"
    echo "Synced mcp.json: color=$color label=$label"
}

wtv-rm() {
    local BRANCH=$1
    local REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
    local DIR=$REPO_ROOT/worktrees/$BRANCH

    if [[ -z "$BRANCH" ]]; then
        echo "Usage: wtv-rm <branch-name>"
        echo ""
        wtv-ls
        return 1
    fi

    if [[ ! -d "$DIR" ]]; then
        echo "Worktree not found: $DIR"
        return 1
    fi

    git worktree remove --force "$DIR" || { echo "Error: could not remove worktree"; return 1; }
    echo "Removed worktree: $DIR"

    # Ask before deleting the branch
    read -r "reply?Delete local branch '$BRANCH' too? [y/N] "
    if [[ "$reply" =~ ^[Yy]$ ]]; then
        git branch -D "$BRANCH" 2>/dev/null && echo "Deleted branch: $BRANCH"
    fi
}

wtv-ls() {
    echo "Active worktrees:"
    git worktree list
    echo ""
    echo "Worktree directories:"
    ls -1 ./worktrees/ 2>/dev/null || echo "(none)"
}

wtv-prune() {
    echo "Pruning stale worktree references..."
    git worktree prune -v
}

wtv-stale() {
    local REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
    local found=0

    echo "Checking for merged worktrees..."
    echo ""
    for dir in "$REPO_ROOT"/worktrees/*/; do
        [[ -d "$dir" ]] || continue
        local branch=$(basename "$dir")
        if git branch --merged master 2>/dev/null | grep -qw "$branch"; then
            echo "  $branch (merged -- safe to remove)"
            found=1
        fi
    done

    if [[ $found -eq 0 ]]; then
        echo "  No stale worktrees found."
    else
        echo ""
        read -r "reply?Remove all stale worktrees? [y/N] "
        if [[ "$reply" =~ ^[Yy]$ ]]; then
            for dir in "$REPO_ROOT"/worktrees/*/; do
                [[ -d "$dir" ]] || continue
                local branch=$(basename "$dir")
                if git branch --merged master 2>/dev/null | grep -qw "$branch"; then
                    git worktree remove --force "$dir" 2>/dev/null
                    git branch -D "$branch" 2>/dev/null
                    echo "  removed $branch"
                fi
            done
        fi
    fi
}
```

## Customization

### Add more files to auto-copy

Extend the `files` array in `wtv()`:

```bash
local files=(
    ".github/prompts/hitl.prompt.md"
    "CLAUDE.md"
    ".vscode/mcp.json"
    ".vscode/settings.json"
    "some/other/file.yaml"   # add here
)
```

### Bash compatibility

The `read` syntax used is for **zsh**. For **bash**, replace:

```bash
# zsh:
read -r "reply?Delete local branch '$BRANCH' too? [y/N] "

# bash:
read -r -p "Delete local branch '$BRANCH' too? [y/N] " reply
```
