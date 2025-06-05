# Git Commands Reference Guide

Here's an overview of all the Git commands we covered in this conversation:

## **Basic Git Workflow**
```bash
git status                    # Check current branch and file status
git add .                     # Stage all changes
git add filename             # Stage specific file
git commit -m "message"      # Commit with message
git push origin branch-name  # Push to remote branch
```

## **Rebase Workflow**
```bash
git fetch origin            # Get latest remote changes
git rebase origin/dev       # Rebase current branch onto dev
git push -f origin branch   # Force push after rebase (when needed)
```

## **Fixing/Undoing Commits**
```bash
git reset --soft HEAD~1     # Undo last commit, keep changes staged
git reset --hard HEAD~1     # Undo last commit, delete changes completely
git reset --hard ORIG_HEAD  # Undo recent rebase/reset
git commit --amend -m "new message"  # Change last commit message
```

## **Interactive Rebase (Advanced)**
```bash
git rebase -i HEAD~2        # Edit last 2 commits interactively
git rebase -i HEAD~3        # Edit last 3 commits interactively
```
**In the editor:**
- `pick` = keep commit as-is
- `squash` = combine with previous commit  
- `reword` = change commit message
- `drop` = delete commit

## **Handling Uncommitted Changes**
```bash
git stash                   # Temporarily save uncommitted changes
git stash pop              # Restore stashed changes
```

## **Emergency Recovery**
```bash
git reflog                 # See history of all Git actions
git reset --hard HEAD@{3}  # Go back to specific point in reflog
```

## **Force Push (Use Carefully!)**
```bash
git push -f origin branch          # Force push (overwrites remote)
git push --force-with-lease origin branch  # Safer force push
```

## **Database & Rails Commands**
```bash
rails db:create             # Create database
rails db:migrate            # Run database migrations
rails console               # Open Rails console
rails db:encryption:init    # Generate encryption keys
```

## **Docker Commands**
```bash
make build                  # Build Docker containers
make shell                  # Enter Docker container shell
docker compose build       # Build containers manually
docker system prune -f     # Clean Docker cache
```

## **Environment Setup**
```bash
# Set database environment variables
export DATABASE_HOST=postgres
export DATABASE_USER=postgres
export DATABASE_PASSWORD=ptogenius_db_pwd
export DATABASE_NAME=ptogenius
export DATABASE_PORT=5432
```

## **Key Rules We Learned:**
- ✅ **Always rebase on feature branches** (cleaner history)
- ✅ **Force push after rebasing** (history changed)
- ❌ **Never force push to shared branches** (main/dev)
- ✅ **Make backup branches before risky operations**
- ✅ **Use `git reflog` if you mess up**
- ✅ **Set environment variables in Docker containers**
- ✅ **Create databases before running migrations**

## **Main Workflow Pattern:**
1. **fetch** → get latest changes from remote
2. **rebase** → replay your commits on top of latest dev
3. **force push** → update remote with rebased history

## **Rebase vs Merge:**
- **Rebase**: Creates linear history, cleaner project timeline
- **Merge**: Preserves branching history, creates merge commits
- **Use rebase for**: Feature branches, personal work
- **Use merge for**: Integrating completed features into main branches

This guide covers the essential Git commands for modern development workflows, with emphasis on maintaining clean commit history through rebasing.