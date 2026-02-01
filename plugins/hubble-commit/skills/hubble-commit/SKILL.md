---
name: hubble-commit
description: Create well-structured git commits with smart file staging, artifact detection, signing, and commit splitting
---

# Hubble Commit Skill

Create well-structured, signed git commits with intelligent file staging, build artifact detection, smart branching decisions, commit splitting analysis, and conventional commit formatting.

## Core Principles

1. **Always sign commits** - Every commit uses `-s` flag for Signed-off-by
2. **No AI attribution** - Never add "Co-Authored-By: Claude" or similar
3. **Focus on WHY** - Commit messages explain why, not what (the diff shows what)
4. **Less detail is better** - Concise messages are more valuable than verbose ones
5. **Small commits** - Prefer multiple focused commits over large omnibus commits

---

## Workflow

Execute these phases in order:

### Phase 1: Repository State Discovery

Gather comprehensive repository state before any decisions:

```bash
# Current branch
CURRENT_BRANCH=$(git branch --show-current)

# Check if on main/master
MAIN_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
IS_ON_MAIN=$([[ "$CURRENT_BRANCH" == "main" || "$CURRENT_BRANCH" == "master" ]] && echo "true" || echo "false")

# Detached HEAD check
IS_DETACHED=$([[ -z "$CURRENT_BRANCH" ]] && echo "true" || echo "false")

# Staged files
STAGED_FILES=$(git diff --cached --name-only)
STAGED_COUNT=0
[[ -n "$STAGED_FILES" ]] && STAGED_COUNT=$(echo "$STAGED_FILES" | wc -l | tr -d ' ')

# Unstaged changes
UNSTAGED_FILES=$(git diff --name-only)
UNSTAGED_COUNT=$(echo "$UNSTAGED_FILES" | grep -c . || echo 0)

# Untracked files
UNTRACKED_FILES=$(git ls-files --others --exclude-standard)
UNTRACKED_COUNT=$(echo "$UNTRACKED_FILES" | grep -c . || echo 0)

# Total lines changed (staged)
LINES_CHANGED=$(git diff --cached --numstat | awk '{sum += $1 + $2} END {print sum+0}')

# Files in staged changes
FILES_CHANGED=$(git diff --cached --stat | tail -1)

# Last commit info (for amend decision)
LAST_COMMIT_SHA=$(git rev-parse HEAD 2>/dev/null)
LAST_COMMIT_MSG=$(git log -1 --format='%s' 2>/dev/null)
LAST_COMMIT_FILES=$(git diff-tree --no-commit-id --name-only -r HEAD 2>/dev/null)

# Check if last commit is pushed
UPSTREAM=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null)
if [[ -n "$UPSTREAM" ]]; then
    LAST_COMMIT_PUSHED=$(git merge-base --is-ancestor HEAD @{upstream} && echo "true" || echo "false")
else
    LAST_COMMIT_PUSHED="unknown"
fi

# Detect if hubble-device-sdk repo
IS_HUBBLE_SDK="false"
if [[ -f "CMakeLists.txt" && -d "src" ]] && git remote -v | grep -q "hubble-device-sdk"; then
    IS_HUBBLE_SDK="true"
fi
# Alternative detection: check for specific SDK files
if [[ -f "include/hubble/hubble.h" || -f "src/hubble_ble.c" ]]; then
    IS_HUBBLE_SDK="true"
fi

# Check for merge conflicts or in-progress merge
MERGE_IN_PROGRESS="false"
if [[ -f ".git/MERGE_HEAD" ]]; then
    MERGE_IN_PROGRESS="true"
fi
CONFLICT_FILES=$(git diff --name-only --diff-filter=U 2>/dev/null)
HAS_CONFLICTS=$([[ -n "$CONFLICT_FILES" ]] && echo "true" || echo "false")

# Check for submodule changes
SUBMODULE_CHANGES=$(git diff --cached --submodule=short 2>/dev/null | grep -c "^Submodule" || echo 0)
```

**Display to user:**
```
Repository State:
- Branch: {CURRENT_BRANCH} {IS_ON_MAIN ? "(main branch)" : ""}
- Staged: {STAGED_COUNT} files ({LINES_CHANGED} lines)
- Unstaged: {UNSTAGED_COUNT} files
- Untracked: {UNTRACKED_COUNT} files
- Last commit: {LAST_COMMIT_MSG} {LAST_COMMIT_PUSHED ? "(pushed)" : "(not pushed)"}
{SUBMODULE_CHANGES > 0 ? "- Submodule changes: " + SUBMODULE_CHANGES : ""}
```

**Early exit conditions:**
- If `IS_DETACHED == true`: Warn user and ask to checkout a branch first
- If `MERGE_IN_PROGRESS == true`: Warn user that a merge is in progress
- If `HAS_CONFLICTS == true`: Warn user to resolve conflicts first and list conflicting files

---

### Phase 2: Untracked and Unstaged File Management

**Trigger:** When `STAGED_COUNT == 0` OR when `UNTRACKED_COUNT > 0` OR when `UNSTAGED_COUNT > 0`

This phase helps users manage their working directory before committing.

#### Step 2a: Artifact Detection

Analyze untracked files for common temporary/build artifact patterns:

```bash
# Common artifact patterns to detect
ARTIFACT_PATTERNS=(
    # Build outputs
    "*.o" "*.a" "*.so" "*.dylib" "*.dll" "*.exe" "*.out"
    "build/" "dist/" "out/" "target/" "bin/" "obj/"

    # Node.js
    "node_modules/"

    # Python
    "__pycache__/" "*.pyc" "*.pyo" "*.pyd" ".eggs/" "*.egg-info/"
    ".venv/" "venv/" "env/" ".env.local"

    # IDE and editor
    ".idea/" ".vscode/settings.json" ".vscode/*.log"
    "*.swp" "*.swo" "*~" ".*.swp"

    # OS files
    ".DS_Store" "Thumbs.db" "desktop.ini"

    # Logs and temp files
    "*.log" "logs/" "*.tmp" "*.temp" "tmp/" "temp/"

    # Coverage and testing
    "coverage/" ".nyc_output/" "htmlcov/" ".coverage"
    ".pytest_cache/" ".tox/"

    # Compiled/generated
    "*.min.js" "*.min.css" "*.map" "*.bundle.*"
)

# Check each untracked file against artifact patterns
LIKELY_ARTIFACTS=""
REGULAR_FILES=""

for file in $UNTRACKED_FILES; do
    IS_ARTIFACT=false

    # Check against patterns using glob matching via case statement
    for pattern in "${ARTIFACT_PATTERNS[@]}"; do
        case "$file" in
            $pattern|*/$pattern)
                IS_ARTIFACT=true
                break
                ;;
        esac
    done

    if [[ "$IS_ARTIFACT" == true ]]; then
        LIKELY_ARTIFACTS="$LIKELY_ARTIFACTS$file\n"
    else
        REGULAR_FILES="$REGULAR_FILES$file\n"
    fi
done
```

#### Step 2b: Handle Detected Artifacts

**If artifacts are detected**, use AskUserQuestion:

```
Potential Build Artifacts Detected

The following untracked files appear to be temporary/build artifacts:
{LIKELY_ARTIFACTS}

These files are typically not committed to version control.

Options:
[G] Add to .gitignore (Recommended)
[S] Stage them anyway
[I] Ignore for now (leave untracked)
```

**If user selects "Add to .gitignore":**

1. Check if `.gitignore` exists in repository root
2. For each artifact pattern that matches files, add to `.gitignore`:

```bash
# Add patterns to .gitignore
for pattern in $DETECTED_ARTIFACT_PATTERNS; do
    if ! grep -qxF "$pattern" .gitignore 2>/dev/null; then
        echo "$pattern" >> .gitignore
    fi
done
```

3. Show what was added:
```
Added to .gitignore:
{list of patterns added}

Would you like to include .gitignore in this commit?
[Y] Yes, stage .gitignore
[N] No, commit separately later
```

#### Step 2c: Handle Regular Untracked Files

**If regular untracked files exist** (after artifact handling), use AskUserQuestion:

```
Untracked Files

The following new files are not yet tracked:
{REGULAR_FILES}

Options:
[A] Add all to staging
[S] Select which to add
[N] None - continue without adding
```

**If user selects "Select which to add":**
- Present a numbered list of files
- Allow user to specify which files to stage (e.g., "1,3,5" or "1-3")
- Stage selected files: `git add <selected_files>`

#### Step 2d: Handle Unstaged Changes

**If unstaged changes exist in tracked files**, use AskUserQuestion:

```
Unstaged Changes

The following tracked files have unstaged changes:
{UNSTAGED_FILES}

Options:
[A] Add all to staging
[S] Select which to add
[N] None - continue without adding
```

**If user selects "Select which to add":**
- Present numbered list
- Stage selected: `git add <selected_files>`

#### Step 2e: Final Staging Check

After file management, re-check staging state:

```bash
STAGED_FILES=$(git diff --cached --name-only)
STAGED_COUNT=0
[[ -n "$STAGED_FILES" ]] && STAGED_COUNT=$(echo "$STAGED_FILES" | wc -l | tr -d ' ')
```

**If `STAGED_COUNT == 0` after file management:**
```
No changes staged for commit.

Nothing to commit. Please stage some changes first with 'git add'.
```
Exit the skill.

---

### Phase 3: Pre-Commit Checks (Conditional)

**Only for hubble-device-sdk repositories:**

```bash
# Check for clang-format issues (use detected main branch)
FORMAT_CHECK=$(git clang-format --verbose --extensions c,h --diff --diffstat "$MAIN_BRANCH" 2>&1)
```

If format issues exist, use AskUserQuestion:

```
Formatting Issues Detected

The following files have formatting issues:
{FORMAT_CHECK}

Options:
[A] Auto-format (run git clang-format)
[M] Manual fix (exit and fix manually)
[S] Skip formatting check
```

If user selects Auto-format:
```bash
git clang-format --verbose --extensions c,h "$MAIN_BRANCH"
```

Show the changes and confirm:
```
Format changes applied:
{show diff}

Options:
[Y] Accept changes and continue
[N] Reject changes and exit
```

---

### Phase 4: Branch Decision

**Trigger:** When `IS_ON_MAIN == true`

Use AskUserQuestion:

```
You're on the main branch. How would you like to proceed?

Options:
[M] Commit directly to main
[B] Create a new branch first
```

If user selects "Create a new branch":

Analyze staged files to suggest branch name:
- Test files -> `test/`
- Documentation -> `docs/`
- Bug fixes (based on file names or patterns) -> `fix/`
- New features -> `feat/`
- Config changes -> `chore/`

```
Suggested branch names based on staged files:
1. feat/{detected-feature-name}
2. fix/{detected-fix-area}
3. [Enter custom name]

Which branch name would you like?
```

Create and switch to the new branch:
```bash
# Modern git uses 'switch -c', fallback to 'checkout -b' for older versions
git switch -c {selected_branch_name} 2>/dev/null || git checkout -b {selected_branch_name}
```

---

### Phase 5: Commit Splitting Analysis

**Trigger:** Large commits meeting ANY of:
- `LINES_CHANGED > 200`
- `STAGED_COUNT > 5`
- Multiple distinct areas detected

Analyze staged files for logical groupings:
- **Test files:** `*_test.*, *_spec.*, test_*, spec_*, tests/*, __tests__/*`
- **Documentation:** `*.md, docs/*, README*`
- **Configuration:** `*.json, *.yaml, *.yml, *.toml, *.ini, .*, Makefile, CMakeLists.txt`
- **Different code areas:** Group by directory or module

Present analysis:

```
Large Commit Detected: {LINES_CHANGED} lines across {STAGED_COUNT} files

Suggested split:
1. feat: core implementation changes (120 lines, 3 files)
   - src/feature.ts
   - src/utils.ts
   - src/types.ts

2. test: add feature tests (80 lines, 2 files)
   - tests/feature.test.ts
   - tests/utils.test.ts

3. docs: update documentation (25 lines, 1 file)
   - README.md

Options:
[S] Split into multiple commits (recommended)
[1] Single commit (all changes together)
[C] Custom split (you specify groupings)
```

If user selects Split:
- Process each group as a separate commit (repeat Phases 7-9 for each)
- Use `git reset HEAD <files>` to unstage files not in current group
- After each commit, re-stage the next group

---

### Phase 6: Amend Decision

**Trigger:** ALL conditions must be true:
- `LAST_COMMIT_PUSHED == false`
- Overlapping files between staged changes and last commit

Calculate file overlap:
```bash
OVERLAP=$(comm -12 <(echo "$STAGED_FILES" | sort) <(echo "$LAST_COMMIT_FILES" | sort))
```

If overlap exists, use AskUserQuestion:

```
Related Unpushed Commit Detected

Your last commit (not yet pushed):
  {LAST_COMMIT_SHA:0:7} - {LAST_COMMIT_MSG}

Files in common with staged changes:
{OVERLAP}

Options:
[A] Amend the previous commit
[N] Create a new commit
```

If amend selected, note to use `--amend` flag in Phase 9.

---

### Phase 7: Commit Type Selection

Analyze staged files to auto-suggest type:
- New files with implementation -> `feat`
- Modified files with bug-related names or fixing patterns -> `fix`
- Test files only -> `test`
- Documentation only -> `docs`
- Config/build files only -> `chore`
- Code restructuring without behavior change -> `refactor`
- Formatting/style changes only -> `style`
- Performance improvements -> `perf`

Use AskUserQuestion:

```
Commit Type

Based on your changes, suggested type: {auto_suggested_type}

Options:
[F] feat - New feature
[X] fix - Bug fix
[R] refactor - Code restructuring
[T] test - Adding/updating tests
[D] docs - Documentation only
[C] chore - Build/config changes
[S] style - Formatting, whitespace
[P] perf - Performance improvement
[O] other - Custom type
```

---

### Phase 8: Commit Message Generation

**Detect Jira Ticket:**
```bash
# Extract from branch name patterns like: feat/HUB-123-description or HUB-456-feature
TICKET=$(echo "$CURRENT_BRANCH" | grep -oE '[A-Z]+-[0-9]+' | head -1)
```

If ticket found, ask:
```
Jira ticket detected: {TICKET}

Include in commit message?
[Y] Yes, include ticket reference
[N] No, omit ticket
```

**Detect Scope:**
Analyze file paths to suggest scope:
- `src/ble/*` -> `ble`
- `src/api/*` -> `api`
- `tests/*` -> detected from implementation files
- Multiple directories -> no scope or ask user

```bash
# Extract scope from staged file paths
STAGED_DIRS=$(echo "$STAGED_FILES" | xargs -n1 dirname 2>/dev/null | sort -u)
SCOPE=""

# Check for common scope patterns
if echo "$STAGED_FILES" | grep -qE "^src/[^/]+/"; then
    # Extract first subdirectory under src/
    SCOPE=$(echo "$STAGED_FILES" | grep -oE "^src/[^/]+" | head -1 | sed 's|src/||')
elif echo "$STAGED_FILES" | grep -qE "^lib/[^/]+/"; then
    SCOPE=$(echo "$STAGED_FILES" | grep -oE "^lib/[^/]+" | head -1 | sed 's|lib/||')
elif echo "$STAGED_FILES" | grep -qE "^packages/[^/]+/"; then
    SCOPE=$(echo "$STAGED_FILES" | grep -oE "^packages/[^/]+" | head -1 | sed 's|packages/||')
fi

# If multiple distinct scopes, leave empty (user can specify)
SCOPE_COUNT=$(echo "$STAGED_DIRS" | wc -l | tr -d ' ')
if [[ "$SCOPE_COUNT" -gt 3 ]]; then
    SCOPE=""
fi
```

**Generate message components:**

1. **Subject line** (required):
   - Format: `{type}({scope}): {description}` or `{type}: {description}`
   - Max 50 characters
   - Imperative mood ("add" not "added", "fix" not "fixed")
   - No period at end
   - Focus on WHY or WHAT at a high level

2. **Body** (optional, for complex changes):
   - Explain motivation and context
   - 72 character line wrap (warn if lines exceed this)
   - Bullet points for multiple items

**Body length validation:**
```bash
# Check if any line in the commit body exceeds 72 characters
validate_body_length() {
    local body="$1"
    local long_lines=$(echo "$body" | awk 'length > 72 {print NR": "substr($0, 1, 50)"..."}')
    if [[ -n "$long_lines" ]]; then
        echo "Warning: The following lines exceed 72 characters:"
        echo "$long_lines"
        return 1
    fi
    return 0
}
```

**Show recent commits for style reference:**
```bash
git log --oneline -5
```

Present draft message:
```
Recent commits for reference:
{recent_commits}

Draft commit message:
---
{type}({scope}): {generated_subject}

{optional_body}
---

Options:
[Y] Use this message
[E] Edit message
[R] Regenerate
```

**Pre-commit warnings** (non-blocking, informational):
```bash
# Check for debug statements (specific patterns to avoid false positives)
# - console.log but not console.error/warn/info
# - debugger statement
# - TODO/FIXME/XXX comments
# - Avoid matching DEBUG in variable names like DEBUG_MODE or isDebugEnabled
DEBUG_PATTERNS="console\.log\(|^\s*debugger;|//\s*(TODO|FIXME|XXX)|#\s*(TODO|FIXME|XXX)"
FOUND_DEBUG=$(git diff --cached -G"$DEBUG_PATTERNS" --name-only)

# Check for potential secrets (simple pattern matching)
SECRET_PATTERNS="password|secret|api_key|apikey|token|credential"
FOUND_SECRETS=$(git diff --cached -G"$SECRET_PATTERNS" -i --name-only)

# Check for large changes (>1000 lines added or removed)
LARGE_FILES=$(git diff --cached --numstat | awk '$1 > 1000 || $2 > 1000 {print $3}')
```

If any found, display warning:
```
Notices:
- Debug statements found in: {FOUND_DEBUG}
- Potential secrets pattern in: {FOUND_SECRETS}
- Large changes in: {LARGE_FILES}

Continue with commit? [Y/N]
```

---

### Phase 9: Commit Execution

**Build the commit command:**

```bash
# Always include -s for Signed-off-by
# NEVER include Co-Authored-By for AI

# Determine if amending
AMEND_FLAG=$([[ "$AMEND_SELECTED" == "true" ]] && echo "--amend" || echo "")

# Check for GPG signing configuration
GPG_SIGN=""
if [[ "$(git config commit.gpgsign)" == "true" ]]; then
    GPG_SIGN="-S"
fi

# Use heredoc for proper message formatting
git commit -s $GPG_SIGN $AMEND_FLAG -m "$(cat <<'EOF'
{type}({scope}): {subject}

{body_if_present}
EOF
)"
```

**Important rules:**
- Always use `-s` flag (adds `Signed-off-by: Name <email>` trailer)
- Never add `Co-Authored-By: Claude` or any AI attribution
- Use heredoc for message to preserve formatting
- If body is empty, use single-line format: `git commit -s -m "{subject}"`

---

### Phase 10: Post-Commit Verification

Show commit summary:
```bash
# Get commit details
COMMIT_SHA=$(git rev-parse HEAD)
COMMIT_MSG=$(git log -1 --format='%s')
COMMIT_STATS=$(git show --stat --format='' HEAD | tail -1)
```

Display:
```
Commit created successfully!

{COMMIT_SHA:0:7} - {COMMIT_MSG}
{COMMIT_STATS}

Signed-off-by: {user_name} <{user_email}>
```

**Show remaining work:**
```bash
REMAINING_UNSTAGED=$(git diff --name-only | wc -l | tr -d ' ')
REMAINING_UNTRACKED=$(git ls-files --others --exclude-standard | wc -l | tr -d ' ')
```

If remaining files exist:
```
Remaining changes:
- {REMAINING_UNSTAGED} unstaged files
- {REMAINING_UNTRACKED} untracked files

Would you like to:
[C] Continue with another commit
[V] View remaining files
[D] Done for now
```

**Suggest next steps:**
```
Next steps:
- Push: git push origin {CURRENT_BRANCH}
- View log: git log --oneline -5
- Run tests: {detected_test_command}
```

---

## Related Files Suggestion

After Phase 1, if unstaged files exist in same directories as staged files:

```bash
# Find directories with staged files
STAGED_DIRS=$(echo "$STAGED_FILES" | xargs -n1 dirname | sort -u)

# Find unstaged files in those directories
for dir in $STAGED_DIRS; do
    RELATED=$(echo "$UNSTAGED_FILES" | grep "^$dir/")
    if [[ -n "$RELATED" ]]; then
        echo "Related unstaged files in $dir: $RELATED"
    fi
done
```

If found:
```
Related unstaged files detected in same directories:
- {file1}
- {file2}

Would you like to stage these too?
[Y] Yes, stage them
[S] Select which to stage
[N] No, continue without them
```

---

## Commit Message Templates

For common patterns, offer templates:

**Bug fix:**
```
fix({scope}): resolve {issue_description}

The {problem} was occurring because {root_cause}.
```

**New feature:**
```
feat({scope}): add {feature_name}

{brief_description_of_what_and_why}
```

**Refactor:**
```
refactor({scope}): {change_description}

No functional changes.
```

---

## Commit History Context

Show last 5 commits to maintain consistent style:
```bash
git log --oneline -5 --format='%s'
```

Use this to match:
- Capitalization style
- Tense usage
- Scope patterns
- Level of detail

---

## Error Handling

**No staged changes:**
```
No changes staged for commit.

Unstaged changes:
{list unstaged files}

Untracked files:
{list untracked files}

Use 'git add <file>' to stage changes.
```

**Pre-commit hook failure:**
```
Pre-commit hook failed. Please fix the issues and try again.

Common fixes:
- Run linter: {detected_lint_command}
- Run formatter: {detected_format_command}
- Fix tests: {detected_test_command}
```

**Detached HEAD:**
```
Warning: You're in detached HEAD state.

Please checkout a branch first:
  git switch -c {suggested_branch_name}
  git switch {existing_branch}
```

**Merge in progress:**
```
Warning: A merge is currently in progress.

Please complete or abort the merge first:
  git merge --continue    # after resolving conflicts
  git merge --abort       # to cancel the merge
```

**Merge conflicts:**
```
Warning: The following files have unresolved merge conflicts:
{CONFLICT_FILES}

Please resolve conflicts before committing:
1. Edit the files to resolve conflicts (look for <<<<<<<, =======, >>>>>>> markers)
2. Stage the resolved files: git add <file>
3. Run this skill again to commit
```

---

## Configuration

This skill respects git configuration:
- `user.name` and `user.email` for Signed-off-by
- `commit.gpgsign` for GPG signing (if enabled, `-S` flag is added)

---

## Verification Checklist

Before finalizing commit:
- [ ] Commit has Signed-off-by trailer (`-s` flag used)
- [ ] No AI attribution in commit (no Co-Authored-By: Claude)
- [ ] Subject line <= 50 characters
- [ ] Subject uses imperative mood
- [ ] Type prefix included (feat, fix, etc.)
- [ ] Body wraps at 72 characters (if present)
- [ ] Commit is signed (if GPG configured)
