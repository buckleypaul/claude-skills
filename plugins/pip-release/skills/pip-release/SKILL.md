---
name: pip-release
description: Package Python libraries and release to PyPI with versioning and changelog
---

# PyPI Release Skill

Package Python libraries for PyPI distribution with automated versioning, changelog generation, and staged release process.

## Core Principles

1. **Always test first** - Never release without passing tests
2. **TestPyPI before production** - Staged release process ensures quality
3. **Sign commits** - Every commit uses `-s` flag for Signed-off-by
4. **No AI attribution** - Never add "Co-Authored-By: Claude" or similar
5. **User approval at each step** - No automated changes without confirmation

---

## Workflow

Execute these phases in order:

### Phase 1: Project Discovery & Validation

Validate the project structure and gather information:

```bash
# Check for pyproject.toml (required)
if [[ ! -f "pyproject.toml" ]]; then
    echo "ERROR: No pyproject.toml found"
    echo "This skill requires a pyproject.toml file for Python packaging."
    echo ""
    echo "To create one, use: python -m pip install build && python -m build --help"
    exit 1
fi

# Parse current version
CURRENT_VERSION=$(python3 -c "
import tomllib
with open('pyproject.toml', 'rb') as f:
    data = tomllib.load(f)
print(data.get('project', {}).get('version', 'NOT_FOUND'))
")

# Parse project name
PROJECT_NAME=$(python3 -c "
import tomllib
with open('pyproject.toml', 'rb') as f:
    data = tomllib.load(f)
print(data.get('project', {}).get('name', 'NOT_FOUND'))
")

# Check for required packaging tools
BUILD_INSTALLED=$(python3 -c "import build; print('yes')" 2>/dev/null || echo "no")
TWINE_INSTALLED=$(python3 -c "import twine; print('yes')" 2>/dev/null || echo "no")

# Detect test infrastructure
HAS_PYTEST=$([[ -f "pytest.ini" || -f "pyproject.toml" ]] && grep -q "pytest" pyproject.toml 2>/dev/null && echo "yes" || echo "no")
HAS_TOX=$([[ -f "tox.ini" ]] && echo "yes" || echo "no")
HAS_UNITTEST=$(find . -name "test_*.py" -o -name "*_test.py" 2>/dev/null | head -1 | grep -q . && echo "yes" || echo "no")

# Git status
CURRENT_BRANCH=$(git branch --show-current)
GIT_STATUS=$(git status --porcelain)
IS_CLEAN=$([[ -z "$GIT_STATUS" ]] && echo "yes" || echo "no")
```

**Display to user:**
```
Project Discovery:
- Project: {PROJECT_NAME}
- Current version: {CURRENT_VERSION}
- Branch: {CURRENT_BRANCH}
- Working tree: {IS_CLEAN ? "clean" : "has uncommitted changes"}

Packaging tools:
- build: {BUILD_INSTALLED}
- twine: {TWINE_INSTALLED}

Test infrastructure:
- tox: {HAS_TOX}
- pytest: {HAS_PYTEST}
- unittest: {HAS_UNITTEST}
```

**Early exit conditions:**
- If `pyproject.toml` doesn't exist: Exit with guidance
- If version is `NOT_FOUND`: Exit with guidance on adding version to pyproject.toml

**Prompt for missing tools:**
```
Missing required packages detected:
- build: {BUILD_INSTALLED == "no" ? "MISSING" : "installed"}
- twine: {TWINE_INSTALLED == "no" ? "MISSING" : "installed"}

Install missing packages?
[Y] Yes, run: pip install build twine
[N] No, exit and install manually
```

---

### Phase 2: Version Detection & Bump

Find the last version change and determine the new version:

```bash
# Find last commit that changed version in pyproject.toml
LAST_VERSION_COMMIT=$(git log --oneline -p -- pyproject.toml | grep -B5 "^+version" | grep "^[a-f0-9]" | head -1 | cut -d' ' -f1)

# Get commits since last version bump
if [[ -n "$LAST_VERSION_COMMIT" ]]; then
    COMMITS_SINCE=$(git log --oneline $LAST_VERSION_COMMIT..HEAD)
    COMMIT_COUNT=$(git rev-list --count $LAST_VERSION_COMMIT..HEAD)
else
    COMMITS_SINCE=$(git log --oneline)
    COMMIT_COUNT=$(git rev-list --count HEAD)
fi
```

**Display commits since last release:**
```
Commits since last release ({COMMIT_COUNT} commits):
{COMMITS_SINCE}
```

**Parse version components:**
```python
import re

version = "{CURRENT_VERSION}"
match = re.match(r'^(\d+)\.(\d+)\.(\d+)(.*)$', version)
if match:
    major, minor, patch, suffix = match.groups()
    major, minor, patch = int(major), int(minor), int(patch)

    patch_version = f"{major}.{minor}.{patch + 1}{suffix}"
    minor_version = f"{major}.{minor + 1}.0{suffix}"
    major_version = f"{major + 1}.0.0{suffix}"
```

**Use AskUserQuestion for version selection:**
```
Version Bump Selection

Current version: {CURRENT_VERSION}
Commits since last release: {COMMIT_COUNT}

Select new version:
[P] Patch: {patch_version} (bug fixes, backwards compatible)
[M] Minor: {minor_version} (new features, backwards compatible)
[J] Major: {major_version} (breaking changes)
[C] Custom version (enter manually)
```

---

### Phase 3: Dependency Verification

Verify declared dependencies match actual imports:

```bash
# Parse declared dependencies from pyproject.toml
DECLARED_DEPS=$(python3 -c "
import tomllib
with open('pyproject.toml', 'rb') as f:
    data = tomllib.load(f)
deps = data.get('project', {}).get('dependencies', [])
for d in deps:
    # Extract package name (before any version specifier)
    import re
    match = re.match(r'^([a-zA-Z0-9_-]+)', d)
    if match:
        print(match.group(1).lower().replace('-', '_'))
")

# Scan imports in source files (find the source directory)
SOURCE_DIR=$(python3 -c "
import tomllib
with open('pyproject.toml', 'rb') as f:
    data = tomllib.load(f)
# Check for src layout
packages = data.get('tool', {}).get('setuptools', {}).get('packages', {})
if 'find' in str(packages):
    where = packages.get('find', {}).get('where', ['.'])
    print(where[0] if isinstance(where, list) else where)
else:
    print('src' if __import__('os').path.isdir('src') else '.')
")

# Get all imports from Python files
ACTUAL_IMPORTS=$(find "$SOURCE_DIR" -name "*.py" -exec grep -h "^import \|^from " {} \; 2>/dev/null | \
    sed 's/^import //' | sed 's/^from //' | \
    cut -d' ' -f1 | cut -d'.' -f1 | \
    sort -u | \
    grep -v "^_" | \
    tr '[:upper:]' '[:lower:]')
```

**Display comparison (informational, non-blocking):**
```
Dependency Analysis (Informational)

Declared dependencies in pyproject.toml:
{DECLARED_DEPS}

External imports found in source:
{ACTUAL_IMPORTS}

Note: Standard library imports and local imports are excluded.
This is for reference only - please verify your dependencies are correct.

Continue with release?
[Y] Yes, continue
[N] No, let me fix dependencies first
```

---

### Phase 4: Changelog Generation

Generate or update the changelog:

```bash
# Check for existing release-notes.md
CHANGELOG_EXISTS=$([[ -f "release-notes.md" || -f "RELEASE-NOTES.md" || -f "CHANGELOG.md" ]] && echo "yes" || echo "no")
CHANGELOG_FILE=$([[ -f "release-notes.md" ]] && echo "release-notes.md" || \
                  [[ -f "RELEASE-NOTES.md" ]] && echo "RELEASE-NOTES.md" || \
                  [[ -f "CHANGELOG.md" ]] && echo "CHANGELOG.md" || echo "release-notes.md")

# Get commits since last version for changelog
COMMITS_FOR_CHANGELOG=$(git log --oneline $LAST_VERSION_COMMIT..HEAD 2>/dev/null || git log --oneline)
```

**Categorize commits by conventional commit type:**
```python
import re
from datetime import date

commits = """
{COMMITS_FOR_CHANGELOG}
"""

categories = {
    'feat': [],
    'fix': [],
    'docs': [],
    'style': [],
    'refactor': [],
    'perf': [],
    'test': [],
    'chore': [],
    'other': []
}

for line in commits.strip().split('\n'):
    if not line:
        continue
    sha, *msg_parts = line.split(' ', 1)
    msg = msg_parts[0] if msg_parts else ''

    # Match conventional commit pattern
    match = re.match(r'^(feat|fix|docs|style|refactor|perf|test|chore)(\(.+\))?:', msg)
    if match:
        category = match.group(1)
        categories[category].append(f"- {msg}")
    else:
        categories['other'].append(f"- {msg}")

# Generate changelog section
changelog_section = f"## [{NEW_VERSION}] - {date.today().isoformat()}\n\n"

category_labels = {
    'feat': 'Added',
    'fix': 'Fixed',
    'docs': 'Documentation',
    'style': 'Style',
    'refactor': 'Refactored',
    'perf': 'Performance',
    'test': 'Tests',
    'chore': 'Maintenance',
    'other': 'Other Changes'
}

for cat, label in category_labels.items():
    if categories[cat]:
        changelog_section += f"### {label}\n"
        changelog_section += '\n'.join(categories[cat])
        changelog_section += '\n\n'
```

**Present draft changelog:**
```
Draft Changelog Entry:

{changelog_section}

Options:
[A] Approve this changelog
[E] Edit changelog (I'll update it)
[S] Skip changelog update
```

---

### Phase 5: README Updates

Find and update version references in README:

```bash
# Find README files
README_FILE=$([[ -f "README.md" ]] && echo "README.md" || \
              [[ -f "README.rst" ]] && echo "README.rst" || \
              [[ -f "readme.md" ]] && echo "readme.md" || echo "")

# Find version references in README
if [[ -n "$README_FILE" ]]; then
    VERSION_REFS=$(grep -n "$CURRENT_VERSION" "$README_FILE" 2>/dev/null || echo "")
fi
```

**Display version references found:**
```
README Version References

File: {README_FILE}

Lines containing current version ({CURRENT_VERSION}):
{VERSION_REFS}

Proposed changes:
- Update version badges
- Update installation examples (pip install {PROJECT_NAME}=={NEW_VERSION})

Options:
[U] Update version references to {NEW_VERSION}
[S] Skip README updates
[M] Let me manually specify which lines to update
```

---

### Phase 6: Pre-Release Summary

Present all planned changes for user approval:

```
Pre-Release Summary

Project: {PROJECT_NAME}
Version: {CURRENT_VERSION} -> {NEW_VERSION}
Branch: {CURRENT_BRANCH}

Files to be modified:
1. pyproject.toml - version bump
2. {CHANGELOG_FILE} - add release notes
3. {README_FILE} - update version references

Commits included in this release ({COMMIT_COUNT}):
{COMMITS_SINCE}

This will:
1. Update version in pyproject.toml
2. Add changelog entry to {CHANGELOG_FILE}
3. Update version references in {README_FILE}
4. Run tests
5. Commit changes with: release: bump version to {NEW_VERSION}
6. Build distribution packages
7. Upload to TestPyPI (required verification)
8. Upload to production PyPI
9. Push to remote and optionally create git tag

Proceed with release?
[Y] Yes, start the release process
[N] No, abort
```

---

### Phase 7: Apply Changes

Apply version bump and changelog updates:

```bash
# Update version in pyproject.toml
# Use Python to properly update TOML
python3 << 'EOF'
import re

with open('pyproject.toml', 'r') as f:
    content = f.read()

# Update version line
content = re.sub(
    r'^version\s*=\s*["\'][^"\']+["\']',
    f'version = "{NEW_VERSION}"',
    content,
    flags=re.MULTILINE
)

with open('pyproject.toml', 'w') as f:
    f.write(content)
EOF

# Update/create changelog
if [[ "$CHANGELOG_EXISTS" == "yes" ]]; then
    # Prepend new section to existing changelog
    python3 << 'EOF'
with open('{CHANGELOG_FILE}', 'r') as f:
    existing = f.read()

# Find insertion point (after header, before first version)
import re
match = re.search(r'^## \[', existing, re.MULTILINE)
if match:
    insert_pos = match.start()
    new_content = existing[:insert_pos] + """{changelog_section}""" + existing[insert_pos:]
else:
    new_content = """{changelog_section}""" + existing

with open('{CHANGELOG_FILE}', 'w') as f:
    f.write(new_content)
EOF
else
    # Create new changelog
    cat > "{CHANGELOG_FILE}" << 'EOF'
# Release Notes

{changelog_section}
EOF
fi

# Update README version references
if [[ -n "$README_FILE" && -f "$README_FILE" ]]; then
    sed -i '' "s/$CURRENT_VERSION/$NEW_VERSION/g" "$README_FILE"
fi
```

**Confirm changes applied:**
```
Changes Applied:

1. pyproject.toml: version updated to {NEW_VERSION}
2. {CHANGELOG_FILE}: release notes added
3. {README_FILE}: version references updated

Review changes:
[D] Show diff of all changes
[C] Continue to tests
[R] Revert all changes and abort
```

---

### Phase 8: Run Tests

Execute the test suite:

```bash
# Detect and run appropriate test command
if [[ "$HAS_TOX" == "yes" ]]; then
    TEST_CMD="tox"
elif [[ "$HAS_PYTEST" == "yes" ]]; then
    TEST_CMD="python -m pytest"
elif [[ "$HAS_UNITTEST" == "yes" ]]; then
    TEST_CMD="python -m unittest discover"
else
    TEST_CMD=""
fi
```

**If tests are found:**
```
Running Tests

Test command: {TEST_CMD}

[Running tests...]
```

```bash
# Execute tests
$TEST_CMD
TEST_EXIT_CODE=$?
```

**If tests fail (BLOCKING):**
```
Tests FAILED

Exit code: {TEST_EXIT_CODE}

Release cannot proceed with failing tests.

Options:
[R] Retry tests
[F] Fix issues and retry
[A] Abort release (revert changes)
```

**If no tests found:**
```
No Test Infrastructure Detected

No tests were found to run. Consider adding tests before release.

Options:
[C] Continue without tests (not recommended)
[A] Abort and add tests first
```

**If tests pass:**
```
Tests PASSED

All tests completed successfully.

Continue to commit changes?
[Y] Yes, commit and continue
[N] No, abort
```

---

### Phase 9: Commit Changes

Create the release commit:

```bash
# Generate commit message with changelog in body
COMMIT_MSG=$(cat << 'EOF'
release: bump version to {NEW_VERSION}

Changes in this release:
{changelog_bullet_points}
EOF
)

# Stage all modified files
git add pyproject.toml
git add "{CHANGELOG_FILE}"
[[ -n "$README_FILE" ]] && git add "$README_FILE"

# Create signed commit (NO Co-Authored-By Claude)
git commit -s -m "$COMMIT_MSG"
```

**Important rules:**
- Always use `-s` flag (adds `Signed-off-by: Name <email>` trailer)
- **NEVER** add `Co-Authored-By: Claude` or any AI attribution
- Include changelog summary in commit body

**Display commit result:**
```
Release Commit Created

Commit: {COMMIT_SHA}
Message: release: bump version to {NEW_VERSION}

Signed-off-by: {user_name} <{user_email}>

Continue to build?
[Y] Yes, build distribution
[N] No, abort (commit will remain)
```

---

### Phase 10: Build Distribution

Build the distribution packages:

```bash
# Clean dist/ directory
rm -rf dist/

# Build distribution
python -m build

# Verify with twine
twine check dist/*
```

**Display build results:**
```
Distribution Built

Files created:
- dist/{PROJECT_NAME}-{NEW_VERSION}.tar.gz
- dist/{PROJECT_NAME}-{NEW_VERSION}-py3-none-any.whl

Twine check: {TWINE_CHECK_RESULT}

Continue to TestPyPI upload?
[Y] Yes, upload to TestPyPI
[N] No, abort
```

---

### Phase 11: TestPyPI Release (Required)

Upload to TestPyPI for verification:

```bash
# Check for authentication
# Priority: env var > .pypirc > prompt

if [[ -n "$TESTPYPI_API_TOKEN" ]]; then
    AUTH_METHOD="environment variable"
    TWINE_USERNAME="__token__"
    TWINE_PASSWORD="$TESTPYPI_API_TOKEN"
elif grep -q "testpypi" ~/.pypirc 2>/dev/null; then
    AUTH_METHOD=".pypirc file"
    # twine will use .pypirc automatically
else
    AUTH_METHOD="manual"
    # Will prompt user
fi
```

**If manual auth needed:**
```
TestPyPI Authentication Required

No TestPyPI credentials found.

Enter your TestPyPI API token:
(Create one at: https://test.pypi.org/manage/account/token/)

Token: [user input]
```

```bash
# Upload to TestPyPI
twine upload --repository testpypi dist/*
```

**Display TestPyPI result:**
```
Uploaded to TestPyPI

Package URL: https://test.pypi.org/project/{PROJECT_NAME}/{NEW_VERSION}/

IMPORTANT: You must verify the package works before continuing!

Verification steps:
1. Create a test virtualenv: python -m venv test-env
2. Activate it: source test-env/bin/activate
3. Install from TestPyPI:
   pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ {PROJECT_NAME}=={NEW_VERSION}
4. Test the package works correctly

Have you verified the package?
[Y] Yes, package works - continue to production
[N] No, abort release
[R] Re-upload to TestPyPI
```

---

### Phase 12: Production PyPI Release

Upload to production PyPI:

```bash
# Check for authentication
if [[ -n "$PYPI_API_TOKEN" ]]; then
    AUTH_METHOD="environment variable"
    TWINE_USERNAME="__token__"
    TWINE_PASSWORD="$PYPI_API_TOKEN"
elif grep -q "\[pypi\]" ~/.pypirc 2>/dev/null; then
    AUTH_METHOD=".pypirc file"
else
    AUTH_METHOD="manual"
fi
```

**If manual auth needed:**
```
PyPI Authentication Required

No PyPI credentials found.

Enter your PyPI API token:
(Create one at: https://pypi.org/manage/account/token/)

Token: [user input]
```

```bash
# Upload to production PyPI
twine upload dist/*
```

**Handle upload errors:**
```
Upload Failed

Error: {ERROR_MESSAGE}

Options:
[R] Retry with new credentials
[A] Abort (package not released)
```

**Display success:**
```
Uploaded to PyPI

Package URL: https://pypi.org/project/{PROJECT_NAME}/{NEW_VERSION}/

Package is now available for installation:
pip install {PROJECT_NAME}=={NEW_VERSION}
```

---

### Phase 13: Push to Remote

Push the release commit and optionally create a git tag:

```bash
# Get current branch and remote
REMOTE=$(git remote | head -1)
BRANCH=$(git branch --show-current)

# Push release commit
git push $REMOTE $BRANCH
```

**Ask about git tag:**
```
Create Git Tag?

A git tag helps mark release points in the repository.

Options:
[Y] Yes, create tag v{NEW_VERSION} and push
[N] No, just push the commit
```

If user selects yes:
```bash
# Create and push tag
git tag -a "v{NEW_VERSION}" -m "Release v{NEW_VERSION}"
git push $REMOTE "v{NEW_VERSION}"
```

---

### Phase 14: Release Summary

Display final summary:

```
Release Complete!

Project: {PROJECT_NAME}
Version: {NEW_VERSION}

Release Artifacts:
- PyPI: https://pypi.org/project/{PROJECT_NAME}/{NEW_VERSION}/
- TestPyPI: https://test.pypi.org/project/{PROJECT_NAME}/{NEW_VERSION}/
- Git commit: {COMMIT_SHA}
- Git tag: v{NEW_VERSION} (if created)

Install command:
pip install {PROJECT_NAME}=={NEW_VERSION}

Files modified:
- pyproject.toml
- {CHANGELOG_FILE}
- {README_FILE}

Next steps:
- Announce the release
- Update documentation
- Close related issues/tickets
- Monitor for user feedback
```

---

## Error Handling

| Error | Action |
|-------|--------|
| No pyproject.toml | Exit with guidance: "This skill requires a pyproject.toml file. Create one with `python -m build --help` or see https://packaging.python.org/tutorials/packaging-projects/" |
| Missing build/twine | Prompt: "Install required packages? pip install build twine" |
| Tests fail | **Block release**, offer retry/abort, suggest fixes |
| Upload fails | Show error, offer retry with new credentials |
| Version parse error | Exit with guidance: "Version must be in format X.Y.Z in pyproject.toml under [project] section" |
| Git dirty state | Warn user, ask to commit or stash changes first |
| Network error | Retry with exponential backoff, then offer manual upload |

---

## Authentication Setup

### Environment Variables (Recommended)
```bash
export PYPI_API_TOKEN="pypi-..."
export TESTPYPI_API_TOKEN="pypi-..."
```

### .pypirc File
```ini
[pypi]
username = __token__
password = pypi-...

[testpypi]
repository = https://test.pypi.org/legacy/
username = __token__
password = pypi-...
```

---

## Configuration

The skill respects:
- `pyproject.toml` for project configuration
- Git configuration for commit signing
- Environment variables for PyPI authentication
- `.pypirc` for PyPI authentication

---

## Verification Checklist

Before completing release:
- [ ] Tests pass
- [ ] Version bumped in pyproject.toml
- [ ] Changelog updated
- [ ] README version references updated
- [ ] Commit has Signed-off-by trailer
- [ ] No AI attribution in commit
- [ ] TestPyPI upload successful
- [ ] User verified TestPyPI package works
- [ ] Production PyPI upload successful
- [ ] Git commit pushed
- [ ] Git tag created (optional)
