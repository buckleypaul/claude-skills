---
name: git-commit-cleanup
description: Clean up messy commit histories by creating a new branch with logical, small, and easily reviewable commits that lead to the same end state.
---

# Git Commit History Cleanup Skill

## Purpose
Clean up messy commit histories by creating a new branch with logical, small, and easily reviewable commits that lead to the same end state.

## When to Use This Skill
- After a development session with many WIP or experimental commits
- When commits are too large or unfocused
- When commit messages are unclear or missing
- Before creating a pull request
- When commits need to be reorganized for better review
- After merging/rebasing that created a messy history

## Core Philosophy: Small, Reviewable Commits

**Prioritize review ease over feature completeness in individual commits.**

Small commits are superior because they:
- Allow reviewers to understand changes incrementally
- Make it easier to identify bugs through git bisect
- Enable safer cherry-picking and reverting
- Reduce cognitive load during code review
- Make it clearer what changed and why
- Facilitate better understanding of the evolution of the code

A single "feature" should typically be broken into multiple commits showing the progression of work.

## Safety & Recovery

### Automatic Backup

Before any cleanup operations begin, **always create a backup tag** to ensure easy recovery:

```bash
# Create backup tag with timestamp
BACKUP_TAG="backup/${CURRENT_BRANCH}-$(date +%Y%m%d-%H%M%S)"
git tag $BACKUP_TAG $CURRENT_BRANCH

# Verify the tag was created
git show-ref --tags $BACKUP_TAG
```

This backup tag preserves the exact state of the original branch, even if the branch is later deleted.

### Recovery Instructions

If anything goes wrong during cleanup:

```bash
# Option 1: Restore branch from backup tag
git checkout -b $CURRENT_BRANCH $BACKUP_TAG

# Option 2: Abort cleanup and delete the new branch
git checkout main
git branch -D $NEW_BRANCH

# Option 3: Hard reset if you accidentally modified the original branch
git checkout $CURRENT_BRANCH
git reset --hard $BACKUP_TAG
```

### Undo Guidance

Present clear undo instructions in the final report:
- Reference the backup tag for full recovery
- Explain how to delete the cleanup branch if unsatisfied
- Remind user that original commits are preserved until they explicitly delete them

## Workflow

### 1. Discovery Phase

```bash
# Identify current branch
CURRENT_BRANCH=$(git branch --show-current)

# Configure base branch (defaults to main, can be overridden)
BASE_BRANCH="${1:-main}"  # Accept as parameter or default to main

# Verify base branch exists
git rev-parse --verify $BASE_BRANCH || { echo "Base branch '$BASE_BRANCH' not found"; exit 1; }

# Find branch point from base
BRANCH_POINT=$(git merge-base $BASE_BRANCH HEAD)

# Get commit count and estimate scope
COMMIT_COUNT=$(git rev-list --count $BRANCH_POINT..HEAD)
echo "Found $COMMIT_COUNT commits to analyze"

# Get list of commits
git log --oneline $BRANCH_POINT..HEAD

# Detect merge commits (require special handling)
MERGE_COMMITS=$(git log --merges --oneline $BRANCH_POINT..HEAD)
if [ -n "$MERGE_COMMITS" ]; then
    echo "⚠️  Warning: Found merge commits that need special handling:"
    echo "$MERGE_COMMITS"
fi

# Detect WIP/fixup commits (candidates for squashing)
WIP_COMMITS=$(git log --oneline $BRANCH_POINT..HEAD | grep -iE '(WIP|fixup|squash|temp|wip:|fix!|FIXME)')
if [ -n "$WIP_COMMITS" ]; then
    echo "📝 Found WIP/fixup commits to merge:"
    echo "$WIP_COMMITS"
fi

# Detect signed commits (signatures won't transfer to new commits)
SIGNED_COMMITS=$(git log --show-signature $BRANCH_POINT..HEAD 2>/dev/null | grep "Good signature" | wc -l)
if [ "$SIGNED_COMMITS" -gt 0 ]; then
    echo "🔐 Warning: Found $SIGNED_COMMITS signed commits. New commits will NOT be signed."
fi

# Check for submodule changes
SUBMODULE_CHANGES=$(git diff --name-only $BRANCH_POINT..HEAD | grep -E '\.gitmodules|^[^/]+$' | head -5)
```

**Partial Cleanup Option:**
For very long-lived branches, you can clean up only recent commits:
```bash
# Clean up only the last N commits
PARTIAL_COUNT=10
BRANCH_POINT=$(git rev-parse HEAD~$PARTIAL_COUNT)
```

### 2. Analysis Phase

Analyze all commits between branch point and HEAD:
- Read each commit's diff to understand what changed
- Identify logical groupings of changes
- Look for opportunities to break large commits into smaller ones
- Identify the overall goal/feature being implemented
- Note any refactoring, formatting, or cleanup changes
- Identify dependencies between changes

### 3. Commit Structure Planning

Design a new commit structure that:

**Size Guidelines:**
- **Aim for 50-200 lines changed per commit** (flexible based on context)
- A single file refactor = one commit
- A new function/class = one commit
- Tests for that function/class = separate commit (or combined if small)
- Configuration changes = separate commit
- Documentation updates = separate commit
- Break large features into logical steps

**Ordering Principles:**
1. Infrastructure/setup changes first
2. Dependencies and utilities
3. Core implementation (broken into small steps)
4. Tests (can be interleaved with implementation)
5. Documentation
6. Configuration/integration
7. Refinements and polish

**Commit Message Quality:**
- Follow conventional commits format: `type(scope): description`
- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `style`, `perf`
- Subject line: imperative mood, 50 chars max, no period
- Body: explain WHAT and WHY, not HOW (72 char wrap)
- Reference issues/tickets if applicable

**Example of Breaking Down a Feature:**
Instead of:
```
feat: implement user authentication
```

Break into:
```
chore: add auth dependencies to package.json
feat: add User model with validation
test: add User model unit tests
feat: add password hashing utility
test: add password hashing tests
feat: implement login endpoint
test: add login endpoint integration tests
feat: implement JWT token generation
test: add JWT token tests
feat: add authentication middleware
test: add auth middleware tests
docs: document authentication API endpoints
```

### 4. Preview & Approval (Dry-Run Mode)

Before implementing, **present the proposed commit structure to the user for approval**. This prevents wasted effort and ensures alignment.

**Present the Plan:**
```
## Proposed Commit Structure

Based on analysis, I propose reorganizing into the following commits:

1. `chore(deps): add authentication dependencies`
   - Files: package.json, package-lock.json
   - Estimated: +45 -2 lines

2. `feat(auth): add User model with validation`
   - Files: src/models/User.ts, src/types/user.ts
   - Estimated: +120 -0 lines

3. `test(auth): add User model unit tests`
   - Files: tests/models/User.test.ts
   - Estimated: +85 -0 lines

... (continue for all proposed commits)

## Dependency Order
- Commit 2 depends on commit 1 (uses new dependencies)
- Commit 3 depends on commit 2 (tests the new model)
- Commits 4-6 can be reordered if preferred

## Questions
- Should tests be combined with implementation or kept separate?
- Any commits you'd like to rename or reorder?
- Ready to proceed with implementation?
```

**User Approval Options:**
- **Approve**: Proceed with implementation as planned
- **Modify**: Adjust commit structure, order, or messages
- **Reject**: Cancel cleanup and keep original branch

**Smart Change Grouping Suggestions:**
During analysis, automatically identify and suggest groupings:
- **Refactoring changes**: Code moves, renames, extractions with no behavior change
- **New feature code**: New files, new functions implementing features
- **Bug fixes**: Changes that correct incorrect behavior
- **Test additions**: New test files or test cases
- **Configuration**: Config files, environment settings
- **Documentation**: Comments, README, docs files

**Dependency Detection:**
Warn if proposed ordering would break dependencies:
```
⚠️  Warning: Proposed commit 3 uses function `validateEmail()`
    which is not introduced until commit 5.
    Suggestion: Move commit 5 before commit 3.
```

### 5. Branch Naming

Generate a logical branch name based on:
- Primary feature or fix being implemented
- Use kebab-case
- Include ticket/issue number if applicable
- Be descriptive but concise
- Pattern: `{type}/{short-description}` or `{type}/{ticket}-{description}`

Examples:
- `feat/user-authentication`
- `fix/ble-connection-timeout`
- `refactor/api-client-cleanup`
- `feat/HUB-123-device-scanner`

### 6. Implementation

```bash
# Create backup tag first (CRITICAL)
BACKUP_TAG="backup/${CURRENT_BRANCH}-$(date +%Y%m%d-%H%M%S)"
git tag $BACKUP_TAG $CURRENT_BRANCH

# Create new branch from base (using configured BASE_BRANCH)
git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH
NEW_BRANCH="feat/cleaned-up-branch-name"
git checkout -b $NEW_BRANCH

# Apply changes as new logical commits
# For each logical commit:
# 1. Make the changes (using git cherry-pick, git apply, or manual editing)
# 2. Stage the specific changes
# 3. Commit with clear message

# Example workflow:
git cherry-pick --no-commit <original-commit-sha>
# Unstage everything
git reset HEAD

# Stage only related changes
git add -p  # Interactive staging for precise control
# Or stage specific files
git add path/to/file1 path/to/file2

# Commit with good message (use -s to add Signed-off-by)
git commit -s -m "feat(auth): add User model with validation

- Create User class with email and password fields
- Add email format validation
- Add password strength requirements
- Include TypeScript type definitions"

# Repeat for each logical commit
```

**Commit Signing:**
All commits use the `-s` flag to add a `Signed-off-by` trailer with the user's configured name and email. This provides clear attribution and is required by many projects (e.g., Linux kernel, CNCF projects).

**Progress Tracking:**
For branches with many commits, report progress during implementation:
```
🔄 Processing commit 1 of 12: chore(deps): add authentication dependencies
✅ Commit 1 complete (+45 -2 lines)
🔄 Processing commit 2 of 12: feat(auth): add User model
✅ Commit 2 complete (+120 -0 lines)
...
```

**File Rename/Move Handling:**
Git can lose rename tracking when changes are split. Keep renames atomic:
```bash
# BAD: Split rename across commits (loses rename detection)
# Commit 1: Delete old file
# Commit 2: Create new file

# GOOD: Keep rename atomic in single commit
git mv old-path/file.ts new-path/file.ts
git commit -s -m "refactor: move file to new location"

# Then in separate commit, make content changes
git commit -s -m "refactor: update file contents"
```

**Submodule Changes:**
Keep submodule pointer updates in separate, clearly labeled commits:
```bash
# Identify submodule changes
git diff --submodule $BRANCH_POINT..HEAD

# Create dedicated commit for submodule updates
git add path/to/submodule
git commit -s -m "chore(submodule): update submodule-name to version X

Updates submodule pointer to include:
- Feature A
- Bug fix B"
```

**Co-authored Commits Preservation:**
When splitting commits that have multiple authors, preserve attribution:
```bash
# Extract co-author info from original commit
CO_AUTHORS=$(git log -1 --format='%(trailers:key=Co-authored-by)' <original-sha>)

# Include in new commit message
git commit -s -m "feat: implement feature

Description of changes

$CO_AUTHORS"
```

**Build Verification Per Commit (Optional):**
For critical branches, verify each commit builds:
```bash
# After each commit, optionally run build
npm run build && npm run lint
# If build fails, fix before proceeding to next commit

# Flag to enable/disable build verification
BUILD_VERIFY=${BUILD_VERIFY:-false}
```

### 7. Verification

```bash
# Verify end states match
git diff $CURRENT_BRANCH..$NEW_BRANCH

# Should output nothing if states match
# If there are differences, investigate and fix

# Review new commit history
git log --oneline $BASE_BRANCH..$NEW_BRANCH

# Review each commit individually
git log -p $BASE_BRANCH..$NEW_BRANCH
```

**Enhanced Diff Verification:**
Beyond basic `git diff`, verify additional file attributes:

```bash
# Verify file permissions match
git diff --stat $CURRENT_BRANCH..$NEW_BRANCH
git ls-tree -r $CURRENT_BRANCH | sort > /tmp/tree-original.txt
git ls-tree -r $NEW_BRANCH | sort > /tmp/tree-new.txt
diff /tmp/tree-original.txt /tmp/tree-new.txt

# Check for symlinks
git ls-files -s | grep ^120000

# Verify no files were accidentally dropped
ORIGINAL_FILES=$(git ls-tree -r --name-only $CURRENT_BRANCH | wc -l)
NEW_FILES=$(git ls-tree -r --name-only $NEW_BRANCH | wc -l)
if [ "$ORIGINAL_FILES" != "$NEW_FILES" ]; then
    echo "⚠️  Warning: File count mismatch! Original: $ORIGINAL_FILES, New: $NEW_FILES"
fi

# Check file permissions haven't changed
git diff --summary $CURRENT_BRANCH..$NEW_BRANCH | grep -E "mode change"
```

**Test Suite Verification (Optional):**
```bash
# If tests exist, run them to verify end state
if [ -f "package.json" ] && grep -q '"test"' package.json; then
    npm test
    if [ $? -ne 0 ]; then
        echo "⚠️  Warning: Tests fail on new branch"
    fi
fi

# Compare test results with original branch
git checkout $CURRENT_BRANCH
npm test > /tmp/tests-original.txt 2>&1
git checkout $NEW_BRANCH
npm test > /tmp/tests-new.txt 2>&1
diff /tmp/tests-original.txt /tmp/tests-new.txt
```

**Commit Count Comparison:**
```bash
# Show before/after metrics prominently
ORIGINAL_COUNT=$(git rev-list --count $BRANCH_POINT..$CURRENT_BRANCH)
NEW_COUNT=$(git rev-list --count $BASE_BRANCH..$NEW_BRANCH)
echo "📊 Commit reduction: $ORIGINAL_COUNT → $NEW_COUNT commits"

# Calculate line changes
git diff --stat $BRANCH_POINT..$CURRENT_BRANCH | tail -1
git diff --stat $BASE_BRANCH..$NEW_BRANCH | tail -1
```

### 8. Reporting

Provide a summary showing:
1. **Old commit structure** - list original commits with message and line changes
2. **New commit structure** - list new commits with message and line changes
3. **Comparison** - highlight how commits were reorganized/split
4. **Verification** - confirm end states match
5. **Next steps** - suggest reviewing the new branch and potentially deleting the old one
6. **Recovery info** - reference backup tag and undo instructions

**Visual Commit Graph:**
Show a simple ASCII visualization of before/after:
```
BEFORE (messy-feature-branch):
* abc1234 - WIP
* def5678 - more fixes
* 111aaaa - fix tests
* 222bbbb - implement feature
* 333cccc - WIP save
* 444dddd - initial attempt
|
* (main)

AFTER (feat/clean-feature):
* aaa1111 - feat(auth): implement authentication
* bbb2222 - test(auth): add authentication tests
* ccc3333 - docs(auth): document auth endpoints
|
* (main)
```

**Commit Mapping Table:**
Show which original commits contributed to which new commits:
```
## Commit Mapping

| New Commit | Original Commits | Description |
|------------|-----------------|-------------|
| aaa1111 | 222bbbb, 333cccc, 444dddd | Core feature implementation |
| bbb2222 | 111aaaa | Test additions |
| ccc3333 | (new) | Documentation (was missing) |

Notes:
- Commits abc1234, def5678 were WIP commits merged into aaa1111
- Documentation commit ccc3333 was created to capture inline comments
```

**Before/After Metrics Summary:**
```
## Cleanup Metrics

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Commits | 47 | 12 | -74% |
| Avg lines/commit | 423 | 156 | -63% |
| Max commit size | 1,847 | 312 | -83% |
| WIP commits | 8 | 0 | -100% |
| Build-breaking commits | 3 | 0 | -100% |
```

**Export to PR Description:**
Generate markdown suitable for a PR description:
```markdown
## Summary
This PR implements user authentication with JWT tokens.

### Commits in this PR
1. **feat(auth): add User model with validation** - Core user data structure
2. **feat(auth): implement JWT token generation** - Token creation and validation
3. **test(auth): comprehensive authentication tests** - Unit and integration tests
4. **docs(auth): API documentation** - Endpoint documentation

### What was changed
- Added 3 new files, modified 2 existing files
- Total: +450 -12 lines

---
*This branch was cleaned up from `messy-feature-branch` (47 commits → 4 commits)*
*Backup available at tag: `backup/messy-feature-branch-20240115-143022`*
```

## Edge Cases to Handle

### No Commits on Branch
If branch has no commits beyond main, inform user and exit gracefully.

### Merge Conflicts
If applying changes causes conflicts:
- Attempt to resolve automatically for simple cases
- For complex conflicts, inform user and provide guidance
- Consider creating commits in a different order to avoid conflicts

### Large Binary Files
- Keep binary file commits separate and atomic
- Don't try to split binary file changes

### Generated Files
- Consider whether generated files (build artifacts, etc.) should be in history
- If they must be included, keep in separate commits
- Add note in commit message that files are generated

### Dependent Changes
- Ensure changes that depend on each other are ordered correctly
- Don't break the build/tests in any individual commit if possible
- If intermediate commits must break tests, note in commit message

### Merge Commits
When the branch contains merge commits, decide how to handle them:

**Option 1: Flatten (Recommended)**
- Treat merged changes as regular commits
- Include all changes from both sides of the merge
- Results in linear history

```bash
# Identify merge commits
git log --merges --oneline $BRANCH_POINT..HEAD

# Cherry-pick with flattening
git cherry-pick --no-commit -m 1 <merge-commit-sha>
```

**Option 2: Preserve Structure**
- Keep merge points visible in history
- Only use when merge history is semantically important

```bash
# Note in commit message that this represents a merge point
git commit -s -m "feat: integrate feature-x branch

This commit combines the work from the feature-x branch merge."
```

### Signed Commits (GPG)
New commits created during cleanup will **NOT** be signed, even if originals were:

```bash
# Detect signed commits before starting
SIGNED_COUNT=$(git log --show-signature $BRANCH_POINT..HEAD 2>/dev/null | grep -c "Good signature")
if [ "$SIGNED_COUNT" -gt 0 ]; then
    echo "⚠️  Warning: Found $SIGNED_COUNT signed commits."
    echo "   New commits will NOT be automatically signed."
    echo "   To sign new commits, configure: git config commit.gpgsign true"
fi
```

If GPG signing is required, ensure GPG is configured and add `-S` flag to commits:
```bash
git commit -s -S -m "feat: signed commit message"
```

Note: `-s` adds Signed-off-by trailer, `-S` adds GPG signature. Both can be used together.

### File Renames and Moves
Git can lose rename tracking when changes are split incorrectly:

**Best Practice:**
1. Keep file moves/renames in their own atomic commit
2. Don't modify file content in the same commit as the rename
3. Verify rename detection after cleanup

```bash
# Check rename detection
git log --follow --oneline -- path/to/file.ts

# If renames aren't detected, consider using:
git log -M --follow -- path/to/file.ts  # -M enables rename detection
```

**Example of Correct Rename Handling:**
```bash
# Commit 1: Rename only
git mv src/old-name.ts src/new-name.ts
git commit -s -m "refactor: rename old-name to new-name"

# Commit 2: Modify content (separate)
# edit src/new-name.ts
git commit -s -m "refactor: update new-name implementation"
```

### Co-authored Commits
When original commits have multiple authors, preserve attribution:

```bash
# Extract co-author trailers from original commit
git log -1 --format='%(trailers:key=Co-authored-by,valueonly)' <sha>

# Include in new commit
git commit -s -m "feat: implement feature

Co-authored-by: Alice <alice@example.com>
Co-authored-by: Bob <bob@example.com>"
```

**Rules for Co-authorship:**
- If splitting a co-authored commit, include co-authors on ALL resulting commits that contain their work
- If combining commits from different authors, include all authors as co-authors
- Original author should remain the commit author; others become co-authors

### Submodule Changes
Handle submodule pointer updates carefully:

```bash
# Identify submodule changes
git diff --submodule=log $BRANCH_POINT..HEAD

# Keep submodule updates in dedicated commits
git add path/to/submodule
git commit -s -m "chore(deps): update submodule-name

Updates from version X to version Y:
- Relevant change 1
- Relevant change 2"
```

**Warning:** Don't split submodule changes across commits - always keep them atomic

## Best Practices

### Commit Message Templates

**Feature commit:**
```
feat(scope): add user authentication system

- Implement JWT-based authentication
- Add login/logout endpoints
- Include token refresh mechanism

Closes #123
```

**Fix commit:**
```
fix(scanner): resolve BLE connection timeout

The connection timeout was occurring because we weren't
properly handling the device discovery completion event.

Modified the scanner to wait for explicit completion signal
before timing out.

Fixes #456
```

**Refactor commit:**
```
refactor(api): extract API client into separate module

No functional changes. Improves code organization and
makes the API client reusable across the application.
```

### Atomic Commit Checklist

Each commit should:
- [ ] Be as small as reasonably possible
- [ ] Represent one logical change
- [ ] Have a clear, descriptive message
- [ ] Build successfully (if applicable)
- [ ] Not break existing tests
- [ ] Be independently reviewable
- [ ] Include related tests (or tests in next commit)

### When to Keep Commits Larger

Sometimes larger commits are appropriate:
- Initial project setup with boilerplate
- Large-scale automated refactoring (but note this in message)
- Vendor library imports
- Mass file renames/moves
- Generated code updates

Even in these cases, consider if there's value in breaking them down.

## Output Format

Present results in this format:

```
## Commit History Cleanup Summary

### Backup Created
Tag: `{BACKUP_TAG}` (preserves original state)

### Original Branch: {CURRENT_BRANCH}
Total commits: {count}
Lines changed: +{additions} -{deletions}

Commits:
1. {sha} - {message} (+{adds} -{dels})
2. {sha} - {message} (+{adds} -{dels})
...

### New Branch: {NEW_BRANCH}
Total commits: {count}
Lines changed: +{additions} -{deletions}

Commits:
1. {sha} - {message} (+{adds} -{dels})
2. {sha} - {message} (+{adds} -{dels})
...

### Cleanup Metrics
| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Commits | {old_count} | {new_count} | {percent}% |
| Avg lines/commit | {old_avg} | {new_avg} | {percent}% |

### Changes Made
- Split commit A into commits X, Y, Z for better reviewability
- Combined commits B and C (both config changes)
- Rewrote all commit messages for clarity
- Reordered commits to show logical progression

### Commit Mapping
| New Commit | Source Commits | Notes |
|------------|---------------|-------|
| {new_sha} | {old_sha1}, {old_sha2} | Combined related changes |
...

### Verification
✓ End states match (git diff shows no differences)
✓ File counts match ({count} files)
✓ All commits build successfully
✓ No commit exceeds 300 lines changed

### Next Steps
1. Review the new commit history: git log {BASE_BRANCH}..{NEW_BRANCH}
2. Compare with original: git log {BASE_BRANCH}..{CURRENT_BRANCH}
3. If satisfied, consider:
   - Pushing the new branch
   - Deleting the old branch (git branch -D {CURRENT_BRANCH})
   - Creating a PR from the new branch

### Recovery (if needed)
If you're not satisfied with the cleanup:
- Restore original branch: `git checkout -b {CURRENT_BRANCH} {BACKUP_TAG}`
- Delete cleanup branch: `git branch -D {NEW_BRANCH}`
- Delete backup tag (after verification): `git tag -d {BACKUP_TAG}`
```

## Configuration Options

The skill can be customized with these parameters:

### Commit Sizing
- `max_commit_size`: Target maximum lines per commit (default: 200, flexible)
- `min_commit_size`: Minimum lines to avoid too-granular commits (default: 10)

### Commit Style
- `commit_style`: Conventional commits format (default: true)
- `include_tests_with_code`: Whether to combine code and tests in same commit (default: false, separate)
- `preserve_wip_commits`: Keep any commits marked as WIP for reference (default: false)
- `sign_off_commits`: Add Signed-off-by trailer using `-s` flag (default: true)
- `gpg_sign_commits`: Add GPG signature using `-S` flag (default: false)

### Branch Configuration
- `base_branch`: Branch to use as base instead of main (default: "main")
- `partial_cleanup_count`: Only clean up the last N commits (default: all commits)

### Safety & Verification
- `create_backup_tag`: Automatically create backup tag before cleanup (default: true)
- `dry_run`: Preview proposed structure without implementing (default: false)
- `verify_build_per_commit`: Run build after each commit (default: false)
- `run_tests_after_cleanup`: Run test suite on new branch (default: false)

### Merge Handling
- `flatten_merges`: Flatten merge commits into linear history (default: true)
- `preserve_co_authors`: Extract and preserve Co-authored-by trailers (default: true)

### Reporting
- `show_commit_mapping`: Include commit mapping table in report (default: true)
- `show_visual_graph`: Show ASCII commit graph visualization (default: true)
- `generate_pr_description`: Generate PR-ready description (default: true)

## Common Patterns

### Pattern: Feature Development
```
chore: add project dependencies
feat: implement core functionality
test: add unit tests for core functionality
feat: add error handling
test: add error handling tests
docs: add API documentation
```

### Pattern: Bug Fix
```
test: add failing test reproducing bug
fix: resolve the bug
test: add additional edge case tests
docs: update documentation with fix notes
```

### Pattern: Refactoring
```
test: add comprehensive tests for current behavior
refactor: extract helper function
refactor: simplify main logic using helper
refactor: rename variables for clarity
docs: update code comments
```

## Notes

- This skill does NOT modify the original branch - it creates a new one
- Always verify the end state matches before deleting original branch
- The new branch is based on the current state of main - ensure main is up to date
- Consider running linting/formatting on the new branch if code style changed
- Small commits may seem like more work, but they dramatically improve review quality

## Success Criteria

A successful cleanup results in:
- ✓ Each commit is focused and small (typically 50-200 lines)
- ✓ Commit messages clearly describe what and why
- ✓ Commits are ordered logically
- ✓ End state exactly matches original branch
- ✓ Each commit is independently reviewable
- ✓ No commit breaks the build or tests
- ✓ The history tells a clear story of how the feature evolved