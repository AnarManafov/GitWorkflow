**Author: Anar Manafov (<Anar.Manafov@gmail.com>)**

---

**Table of Contents**

- [Introduction](#introduction)
- [Quick Reference](#quick-reference)
- [Prerequisites and Best Practices](#prerequisites-and-best-practices)
- [The Workflow](#the-workflow)
  - [Core Principles](#core-principles)
  - [Branch Types](#branch-types)
    - [master branch](#master-branch)
    - [develop branch](#develop-branch)
    - [release branch](#release-branch)
    - [hotfix branch](#hotfix-branch)
    - [feature branches](#feature-branches)
  - [Team Roles](#team-roles)
- [Getting Started](#getting-started)
  - [Initial Setup](#initial-setup)
- [Developer Workflows](#developer-workflows)
  - [Working on Features](#working-on-features)
  - [Keeping Your Branch Updated](#keeping-your-branch-updated)
  - [Submitting Your Work](#submitting-your-work)
- [Release Manager Workflows](#release-manager-workflows)
  - [Processing Pull Requests](#processing-pull-requests)
  - [Creating Releases](#creating-releases)
  - [Managing Hotfixes](#managing-hotfixes)
- [Advanced Topics](#advanced-topics)
  - [Recovering from Upstream Rebases](#recovering-from-upstream-rebases)
  - [Working with Patches](#working-with-patches)
- [Modern Git Features](#modern-git-features)

# Introduction

This document describes a **rebase-based Git workflow** designed for development teams who prioritize clean history and stable releases. This workflow provides:

- **Uninterrupted development** - Multiple developers can work simultaneously without blocking each other
- **Always-stable master branch** - The master branch is always ready for production deployment
- **Multi-layered conflict prevention** - Conflicts are resolved early in feature branches
- **Error recovery at multiple levels** - Mistakes can be caught and fixed before reaching production
- **Clean, linear history** - No merge commits cluttering the master branch timeline

> [!IMPORTANT]
> **Core Philosophy**: Treat public history as immutable and easy to follow. Treat private history as disposable and malleable.

---

# Quick Reference

## Common Commands

```bash
# Start working on a new feature
git switch -c feature/ABC-123 upstream/develop

# Keep your feature branch updated
git fetch upstream
git rebase upstream/develop

# Submit your work (after squashing commits)
git push --force-with-lease origin feature/ABC-123

# Clean up after merge
git branch -d feature/ABC-123
git push origin --delete feature/ABC-123
```

## Branch Naming Conventions

- **Features**: `feature/ABC-123-short-description`
- **Hotfixes**: `hotfix/v1.2.3-critical-fix`
- **Releases**: `release/v1.3.0`

---

# Prerequisites and Best Practices

## Development Guidelines

- **One branch per feature/bug** - Each ticket/task must be implemented on a separate branch
- **Meaningful branch names** - Use descriptive names that include ticket numbers (e.g., `feature/ABC-123-user-authentication`)
- **Regular synchronization** - Rebase your feature branch frequently to avoid large conflicts
- **Clean commit history** - Squash related commits before submitting for review
- **Atomic commits** - Each commit should represent a single logical change

## Repository Hygiene

- **No large repositories** - Keep repositories focused and manageable in size
- **Use Git LFS** - For large binary files, enable [Git LFS](https://git-lfs.com)
- **Exclude generated files** - Never commit files that can be regenerated (build artifacts, IDE files, etc.)
- **Write access control** - Only release managers should have write access to `master` and `develop` branches

## Git Configuration

Set up these configurations for all team members:

```bash
# Enable automatic rebase for new branches
git config --global branch.autosetuprebase always

# Set your identity
git config --global user.name "Your Full Name"
git config --global user.email "your.email@company.com"

# Case-sensitive file names (important for cross-platform work)
git config --global core.ignorecase false

# Better merge conflict resolution
git config --global merge.conflictstyle diff3

# Default branch name (optional - can keep as main for new repos)
git config --global init.defaultBranch master
```

---

# The Workflow

```mermaid
graph TD
    M1[master: v1.0.0] --> D1[develop]
    D1 --> F1[feature/ABC-123]
    F1 --> F2[Work & commits]
    F2 --> F3[Rebase from develop]
    F3 --> F4[Squash commits]
    F4 --> D2[develop: merge FF]
    
    D2 --> R1[release/v1.1.0]
    R1 --> R2[Bug fixes only]
    R2 --> M2[master: v1.1.0]
    M2 --> D3[develop: rebase]
    
    M1 --> H1[hotfix/v1.0.1]
    H1 --> H2[Critical fix]
    H2 --> M3[master: v1.0.1]
    M3 --> D4[develop: rebase]
    
    style M1 fill:#ffcdd2
    style M2 fill:#ffcdd2
    style M3 fill:#ffcdd2
    style D1 fill:#c8e6c9
    style D2 fill:#c8e6c9
    style D3 fill:#c8e6c9
    style D4 fill:#c8e6c9
    style F1 fill:#e1bee7
    style R1 fill:#fff3e0
    style H1 fill:#ffecb3
```

## Branch Flow Visualization

```mermaid
flowchart LR
    subgraph "Developer Workflow"
        DEV[👨‍💻 Developer] --> CREATE[Create Feature Branch]
        CREATE --> WORK[Develop Feature]
        WORK --> REBASE[Rebase from develop]
        REBASE --> SQUASH[Squash Commits]
        SQUASH --> PR[Create Pull Request]
    end
    
    subgraph "Release Manager Workflow"
        PR --> REVIEW[🔍 Review PR]
        REVIEW --> MERGE[Fast-Forward Merge]
        MERGE --> RELEASE{Ready for Release?}
        RELEASE -->|Yes| RC[Create Release Branch]
        RELEASE -->|No| CREATE
        RC --> TEST[Testing & Bug Fixes]
        TEST --> PROD[Deploy to Production]
        PROD --> TAG[🏷️ Tag Release]
        TAG --> CLEANUP[Clean up branches]
    end
    
    subgraph "Hotfix Process"
        ISSUE[🚨 Critical Issue] --> HOTFIX[Create Hotfix Branch]
        HOTFIX --> FIX[Implement Fix]
        FIX --> DEPLOY[Deploy Hotfix]
        DEPLOY --> HTAG[🏷️ Tag Hotfix]
        HTAG --> SYNC[Sync develop]
    end
    
    style DEV fill:#e3f2fd
    style REVIEW fill:#fff3e0
    style ISSUE fill:#ffebee
```

## Core Principles

Understanding the difference between these types of changes is crucial for following the correct workflow path:

- **Features & Bug Fixes**: Changes that can wait for the next scheduled release
  - Path: `feature branch` → `develop` → `release branch` → `master`
  - These follow the full development cycle with proper testing and review

- **Hotfixes**: Critical fixes that cannot wait for the next release
  - Path: `hotfix branch` (from production tag) → `master`
  - These bypass the normal development cycle for urgent production issues

## Branch Types

### master branch

The **master branch** contains only stable, production-ready code.

**Characteristics:**
- All production releases are tagged here
- Always buildable and deployable
- No direct development allowed
- Only release managers have write access
- History moves only forward (no force pushes)
- New changes enter only via fast-forward merges

**Protection Rules:**
```bash
# Only fast-forward merges allowed
git merge --ff-only feature-branch
```

### develop branch

The **develop branch** is the main development integration point.

**Characteristics:**
- Branched from the latest `master`
- Integration point for all feature branches
- Periodically rebased from `master` when releases are completed
- Contains the latest development changes destined for the next release

**Maintenance:**
```bash
# Keep develop updated with master after each release
git switch develop
git rebase master
```

### release branch

**Release branches** are temporary branches for release preparation.

**Purpose:**
- Enables feature freeze while allowing continued development
- Provides isolated environment for release testing and bug fixes
- Only critical bug fixes allowed (no new features)

**Lifecycle:**
1. Created from `develop` when ready for release
2. Bug fixes applied during testing phase
3. Merged to `master` when release is complete
4. Deleted after successful merge

**Example:**
```bash
# Create release branch
git switch -c release/v1.3.0 develop

# After release is complete
git switch master
git merge --ff-only release/v1.3.0
git tag v1.3.0
git branch -d release/v1.3.0
```

### hotfix branch

**Hotfix branches** handle critical production issues.

**Characteristics:**
- Created from the affected production tag on `master`
- Contains only the minimal fix for the critical issue
- Merged directly back to `master`
- `develop` branch must be rebased after hotfix integration

**Example:**
```bash
# Create hotfix from production tag
git switch -c hotfix/v1.2.3-critical-fix v1.2.2

# After fix is complete and tested
git switch master
git merge --ff-only hotfix/v1.2.3-critical-fix
git tag v1.2.3

# Update develop with the hotfix
git switch develop
git rebase master
```

### feature branches

**Feature branches** are where individual development work happens.

**Naming Convention:**
- `feature/TICKET-123-short-description`
- `bugfix/TICKET-456-fix-login-issue`

**Best Practices:**
- Created from latest `develop`
- Regularly rebased against `develop`
- Commits squashed before integration
- Deleted after successful merge

## Team Roles

| Role | Permissions | Responsibilities |
|------|-------------|------------------|
| **Developer** | Read: `master`, `develop`<br>Write: `feature/*`, `bugfix/*` | Feature development, bug fixes, code reviews |
| **Release Manager** | Read/Write: All branches | Integration, releases, hotfixes, branch management |

---

# Getting Started

## Initial Setup

This setup process applies to both developers and release managers.

### 1. Configure Git

Set up your Git configuration (run once per machine):

```bash
# Enable automatic rebase for new branches
git config --global branch.autosetuprebase always

# Set your identity (use your real name and email)
git config --global user.name "John Doe"
git config --global user.email "john.doe@company.com"

# Case-sensitive file names
git config --global core.ignorecase false

# Better merge conflict resolution
git config --global merge.conflictstyle diff3
```

### 2. Fork and Clone

1. **Fork the main repository** on GitHub/GitLab
2. **Clone your fork locally**:

```bash
git clone https://github.com/YOUR_USERNAME/PROJECT_NAME.git
cd PROJECT_NAME
```

### 3. Set Up Remotes

Add the main repository as your upstream remote:

```bash
# Add upstream remote (main repository)
git remote add upstream https://github.com/ORGANIZATION/PROJECT_NAME.git

# Fetch all branches
git fetch upstream

# Verify your remotes
git remote -v
# Should show:
# origin    https://github.com/YOUR_USERNAME/PROJECT_NAME.git (fetch)
# origin    https://github.com/YOUR_USERNAME/PROJECT_NAME.git (push)
# upstream  https://github.com/ORGANIZATION/PROJECT_NAME.git (fetch)
# upstream  https://github.com/ORGANIZATION/PROJECT_NAME.git (push)
```

### 4. Create Local Development Branch

```bash
# Create and switch to local develop branch
git switch -c develop upstream/develop

# Push to your fork and set up tracking
git push -u origin develop
```

---

# Developer Workflows

## Working on Features

### 1. Create a Feature Branch

Always create feature branches from the latest `develop`:

```bash
# Fetch latest changes
git fetch upstream

# Create feature branch from upstream develop
git switch -c feature/ABC-123-user-authentication upstream/develop

# Push to your fork and set up tracking
git push -u origin feature/ABC-123-user-authentication
```

**Branch Naming Guidelines:**

- Include ticket number: `feature/ABC-123-description`
- Use lowercase with hyphens: `feature/abc-123-user-auth`
- Keep description short but meaningful
- For bugs: `bugfix/DEF-456-login-crash`

### 2. Develop Your Feature

Make commits as you work, following these practices:

```bash
# Make your changes
# ... edit files ...

# Stage and commit
git add .
git commit -m "Add user authentication endpoint

- Implement JWT token generation
- Add password validation
- Include rate limiting

Closes ABC-123"
```

**Commit Message Guidelines:**

- First line: brief summary (50 chars max)
- Blank line, then detailed description
- Include ticket references
- Use imperative mood ("Add" not "Added")

## Keeping Your Branch Updated

Regularly sync your feature branch with `develop` to avoid conflicts:

```bash
# Fetch latest changes
git fetch upstream

# Switch to your feature branch
git switch feature/ABC-123-user-authentication

# Rebase onto latest develop
git rebase upstream/develop
```

**If conflicts occur:**

```bash
# Git will pause rebase and show conflicts
# Edit files to resolve conflicts, then:

git add <resolved-file>
git rebase --continue

# If you need to abort the rebase:
git rebase --abort
```

**Push your updated branch:**

```bash
# After successful rebase, force push (safely)
git push --force-with-lease origin feature/ABC-123-user-authentication
```

> **Why `--force-with-lease`?** This is safer than `--force` because it checks that no one else has pushed to your branch since your last fetch.

## Submitting Your Work

### 1. Final Preparation

Before submitting, ensure your branch is clean and up-to-date:

```bash
# Fetch and rebase one final time
git fetch upstream
git rebase upstream/develop

# Review your commits
git log --oneline upstream/develop..HEAD
```

### 2. Squash Your Commits

Clean up your commit history by squashing related commits:

```bash
# Interactive rebase to squash commits
git rebase -i upstream/develop
```

In the interactive editor:

- Keep the first commit as `pick`
- Change others to `squash` or `s`
- Write a clear, comprehensive commit message

**Example squash result:**

```
Add user authentication system

- Implement JWT token generation and validation
- Add bcrypt password hashing
- Include rate limiting for login attempts  
- Add comprehensive unit tests
- Update API documentation

Closes ABC-123
```

### 3. Create Pull Request

```bash
# Push your final version
git push --force-with-lease origin feature/ABC-123-user-authentication
```

Then create a pull request from your fork to the main repository's `develop` branch.

**Pull Request Guidelines:**

- **Title**: `[ABC-123] Add user authentication system`
- **Description**: Explain what was implemented and why
- **Testing**: Describe how to test the changes
- **Screenshots**: Include for UI changes

### 4. Stop Working on the Branch

> [!IMPORTANT]
> After submitting your pull request, **stop working on that feature branch**. Start a new branch for any additional work.

This prevents complications during the review and merge process.

---

# Release Manager Workflows

Release managers have write access to the main repository and are responsible for integrating changes and managing releases.

## Processing Pull Requests

### Modern Approach: Platform Integration

Most Git platforms (GitHub, GitLab) provide built-in support for rebase-based workflows:

**GitHub:**
- Use "Rebase and merge" option for pull requests
- Enable "Require linear history" branch protection
- Set up automated checks before allowing merge

**GitLab:**
- Configure "Fast-forward merge" in project settings
- Use merge request pipelines for automated testing

### Manual Processing (if needed)

If you need to process pull requests manually:

```bash
# 1. Update your local repository
git fetch origin
git fetch upstream

# 2. Add developer's repository (first time only)
git remote add dev-john https://github.com/john/PROJECT_NAME.git
git fetch dev-john

# 3. Switch to develop and ensure it's current
git switch develop
git rebase upstream/develop

# 4. Attempt fast-forward merge
git merge --ff-only dev-john/feature/ABC-123-user-auth
```

**If fast-forward fails:**
- Request the developer to rebase their branch and resubmit
- The merge should always be fast-forward only

```bash
# 5. Push to main repository (if merge succeeded)
git push upstream develop
```

## Creating Releases

### 1. Create Release Branch

When `develop` contains all features for the next release:

```bash
# Create release branch from develop
git switch -c release/v1.3.0 develop

# Push to main repository
git push upstream release/v1.3.0
```

### 2. Release Testing and Bug Fixes

During the release testing phase:

- Only critical bug fixes are allowed on the release branch
- No new features
- Developers continue working on `develop` for the next release

```bash
# Example: Apply bug fix to release branch
git switch release/v1.3.0
git merge --ff-only hotfix-branch-from-developer
```

### 3. Complete the Release

When testing is complete and the release is ready:

```bash
# 1. Switch to master and merge release
git switch master
git merge --ff-only release/v1.3.0

# 2. Tag the release
git tag -a v1.3.0 -m "Release version 1.3.0

- Feature: User authentication system
- Feature: New dashboard layout  
- Fix: Memory leak in data processor
- Fix: Login timeout issues"

# 3. Push to main repository
git push upstream master
git push upstream v1.3.0

# 4. Update develop with any release fixes
git switch develop
git rebase master

# 5. Clean up release branch
git branch -d release/v1.3.0
git push upstream --delete release/v1.3.0
```

## Managing Hotfixes

### 1. Create Hotfix Branch

For critical production issues:

```bash
# Create hotfix from the affected production tag
git switch -c hotfix/v1.2.3-security-patch v1.2.2

# Make the minimal fix
# ... edit files ...
git add .
git commit -m "Fix critical security vulnerability

- Sanitize user input in login form
- Add input validation tests
- Update security documentation

Fixes: SEC-789"
```

### 2. Test and Deploy Hotfix

Test the hotfix thoroughly in an isolated environment before proceeding.

### 3. Integrate Hotfix

```bash
# 1. Merge to master
git switch master  
git merge --ff-only hotfix/v1.2.3-security-patch

# 2. Tag the hotfix release
git tag -a v1.2.3 -m "Hotfix v1.2.3: Security patch"

# 3. Push to main repository
git push upstream master
git push upstream v1.2.3

# 4. Update develop
git switch develop
git rebase master

# 5. Clean up hotfix branch
git branch -d hotfix/v1.2.3-security-patch
```

---

# Advanced Topics

## Recovering from Upstream Rebases

Sometimes the upstream branch history needs to be changed (rebased, commits moved/deleted/squashed). When this happens, feature branches derived from the old history will have problems rebasing.

### The Problem

**Before upstream rebase:**

```text
C0 ← C1 ← C2 ← C3 ← C4    (develop)
          ↑
          └─ cf1 ← cf2     (your feature branch)
```

**After upstream rebase:**

```text
C0 ← C1x ← C2x ← C3x ← C4x (develop - rebased)

C0 ← C1 ← C2 ← cf1 ← cf2   (your feature branch - now orphaned)
```

Git sees `C1`, `C2` as different from `C1x`, `C2x`, causing merge conflicts when rebasing.

### The Solution

Use `git rebase --onto` to move your commits to the new base:

```bash
# Fetch the updated upstream
git fetch upstream

# Rebase your feature commits onto the new develop
git rebase --onto upstream/develop develop feature/ABC-123
```

This command means: "Take commits from `develop` to `feature/ABC-123` and replay them onto `upstream/develop`."

**Alternative approach if you know the commit IDs:**

```bash
# Find the old parent commit ID
OLD_PARENT=$(git merge-base develop upstream/develop)

# Rebase onto new parent  
git rebase --onto upstream/develop $OLD_PARENT feature/ABC-123
```

### Prevention

To minimize the need for history changes:

- Keep feature branches short-lived
- Rebase frequently to stay current
- Squash commits before submitting pull requests

## Working with Patches

Sometimes you may need to work with patches instead of direct git operations.

### Creating Patches

```bash
# Create patch for all commits different from master
git format-patch upstream/master --stdout > my-feature.patch

# Create patch for specific commits
git format-patch HEAD~3..HEAD --stdout > last-3-commits.patch

# Create patch for specific files
git format-patch HEAD~1 --stdout -- src/specific-file.js > file-changes.patch
```

### Applying Patches

```bash
# Apply patch with sign-off
git am --signoff < my-feature.patch

# Apply with different line ending handling
git am --ignore-space-change --ignore-whitespace --signoff < my-feature.patch

# Preserve carriage returns (Windows)
git am --keep-cr --signoff < my-feature.patch

# If patch fails to apply cleanly
git am --abort  # to cancel
# or
git am --skip   # to skip current patch
```

---

# Modern Git Features

This workflow can benefit from several modern Git features and tools.

## Modern Commands

**Use `git switch` instead of `git checkout` for branch operations:**

```bash
# Old way
git checkout -b feature/new-feature

# Modern way  
git switch -c feature/new-feature

# Switch to existing branch
git switch develop
```

**Use `git restore` for file operations:**

```bash
# Old way
git checkout -- file.txt

# Modern way
git restore file.txt

# Restore from specific commit
git restore --source=HEAD~2 file.txt
```

## Enhanced Safety

**Use `--force-with-lease` with additional safety:**

```bash
# Even safer than --force-with-lease
git push --force-if-includes origin feature/ABC-123
```

**Configure safer defaults:**

```bash
# Require confirmation for force pushes
git config --global push.default simple
git config --global push.followTags true

# Better diff handling
git config --global diff.algorithm histogram
git config --global merge.conflictstyle diff3
```

## Productivity Enhancements

**Auto-stash during rebase:**

```bash
# Automatically stash/unstash during rebase
git pull --rebase --autostash upstream develop
```

**Better branch management:**

```bash
# List branches and their status
git branch -vv

# Find merged branches
git branch --merged upstream/master

# Clean up merged branches
git branch --merged upstream/master | grep -v master | xargs git branch -d
```

## Integration with Modern Tools

### GitHub CLI

```bash
# Install GitHub CLI
brew install gh  # macOS
# or download from https://cli.github.com

# Create PR from command line
gh pr create --title "Add user authentication" --body "Implements JWT-based auth system"

# Review PRs
gh pr list
gh pr view 123
gh pr checkout 123
```

### Improved Workflow Commands

```bash
# View commit graph
git log --graph --oneline --all

# Interactive branch switching
git switch $(git branch | fzf)  # requires fzf

# Better blame with ignore revisions
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

## Platform-Specific Features

### GitHub Features

- **Draft Pull Requests**: Mark PRs as work-in-progress
- **Auto-merge**: Automatically merge when checks pass
- **Merge Queues**: Serialize merges to prevent conflicts
- **Required Status Checks**: Enforce CI/CD before merge

### GitLab Features

- **Merge Request Pipelines**: Run CI on merge result
- **Semi-linear Merge**: Rebase with merge commit
- **Push Rules**: Enforce commit message formats

---

## Conclusion

This rebase-based workflow provides:

✅ **Clean History**: Linear commit history without merge pollution  
✅ **Stable Master**: Always-deployable master branch  
✅ **Parallel Development**: Teams can work without blocking each other  
✅ **Quality Control**: Multiple checkpoints before production  
✅ **Modern Tooling**: Leverages latest Git and platform features  

The key to success is **consistent application** of these practices by all team members and **regular communication** between developers and release managers.

For questions or suggestions, contact: [Anar.Manafov@gmail.com](mailto:Anar.Manafov@gmail.com)
