# Agent Thinking Skills Wrapper Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the forked thinking-skills collection installable from both Claude and Codex marketplaces while preserving upstream skill content.

**Architecture:** Keep the upstream root `skills/` directory unchanged. Add Codex plugin metadata, agent governance notes, and a scheduled upstream sync workflow that opens reviewable pull requests.

**Tech Stack:** Claude plugin manifest JSON, Codex plugin manifest JSON, GitHub Actions, Node-based upstream skill validator, marketplace `url` sources.

---

## File Structure

- Read: `.claude-plugin/plugin.json`
  - Existing Claude plugin manifest to preserve.
- Create: `.codex-plugin/plugin.json`
  - Codex plugin manifest for the fork.
- Create: `AGENTS.md`
  - Agent guidance for vendored upstream skill content.
- Create: `.github/workflows/sync-upstream.yml`
  - Scheduled/manual workflow that syncs from `tjboudreaux/cc-thinking-skills`.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.claude-plugin/marketplace.json`
  - Point the Claude marketplace entry at the maintained fork if required.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.agents/plugins/marketplace.json`
  - Point the Codex marketplace entry at the maintained fork.

## Task 1: Add Codex Plugin Manifest

**Files:**
- Create: `.codex-plugin/plugin.json`
- Read: `.claude-plugin/plugin.json`

- [ ] **Step 1: Inspect the existing Claude manifest**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json
```

Expected: valid JSON with plugin name `thinking-skills`.

- [ ] **Step 2: Create Codex manifest directory**

Run:

```bash
mkdir -p .codex-plugin
```

Expected: command exits 0.

- [ ] **Step 3: Add `.codex-plugin/plugin.json`**

Create `.codex-plugin/plugin.json` with:

```json
{
  "name": "thinking-skills",
  "version": "1.0.0",
  "description": "39 mental models and critical thinking frameworks packaged for Codex marketplace installation.",
  "author": {
    "name": "TJ Boudreaux",
    "url": "https://github.com/tjboudreaux"
  },
  "repository": "https://github.com/accelerate-data/agent-thinking-skills"
}
```

If the fork is moved to an Accelerate Data-owned repository before
implementation, use that repository URL instead.

- [ ] **Step 4: Validate JSON**

Run:

```bash
python3 -m json.tool .codex-plugin/plugin.json
```

Expected: pretty-printed JSON and exit 0.

- [ ] **Step 5: Commit**

Run:

```bash
git add .codex-plugin/plugin.json
git commit -m "feat: add Codex plugin manifest"
```

Expected: commit succeeds.

## Task 2: Add Agent Guidance for Upstream-Owned Skills

**Files:**
- Create: `AGENTS.md`

- [ ] **Step 1: Create `AGENTS.md`**

Create `AGENTS.md` with:

```markdown
# AGENTS.md

This repository is a maintained fork of `tjboudreaux/cc-thinking-skills`.

## Vendored Upstream Content

The skill content under `skills/` is upstream-owned content from
`tjboudreaux/cc-thinking-skills`.

AI agents may edit wrapper-owned files such as `.codex-plugin/`,
`.github/workflows/`, documentation, and validation scripts as normal.

Before editing files under `skills/`, AI agents must warn the user that the file is
upstream-owned and explain that the change may diverge from future upstream syncs.
Proceed only when the user explicitly confirms the local divergence or when the task
is specifically to resolve an upstream sync conflict.

## Repository Purpose

This repository packages the upstream thinking skills collection for marketplace
installation in Claude and Codex. It should preserve upstream skill behavior unless
a reviewed local divergence is explicitly requested.
```

- [ ] **Step 2: Review the rendered guidance**

Run:

```bash
sed -n '1,120p' AGENTS.md
```

Expected: the vendored-content warning is present and clear.

- [ ] **Step 3: Commit**

Run:

```bash
git add AGENTS.md
git commit -m "docs: add vendored skill guidance"
```

Expected: commit succeeds.

## Task 3: Add Scheduled Upstream Sync Workflow

**Files:**
- Create: `.github/workflows/sync-upstream.yml`

- [ ] **Step 1: Create workflow directory**

Run:

```bash
mkdir -p .github/workflows
```

Expected: command exits 0.

- [ ] **Step 2: Add workflow**

Create `.github/workflows/sync-upstream.yml` with:

```yaml
name: Sync upstream thinking skills

on:
  schedule:
    - cron: "0 9 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout fork
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: git remote add upstream https://github.com/tjboudreaux/cc-thinking-skills.git

      - name: Fetch upstream
        run: git fetch upstream main

      - name: Create sync branch
        run: git checkout -B sync/upstream-thinking-skills

      - name: Merge upstream
        run: git merge upstream/main --no-edit

      - name: Validate wrapper files
        run: |
          python3 -m json.tool .claude-plugin/plugin.json >/dev/null
          python3 -m json.tool .codex-plugin/plugin.json >/dev/null
          test -d skills
          find skills -name SKILL.md -type f | grep -q .
          node scripts/validate-skills.js

      - name: Push sync branch
        run: git push origin sync/upstream-thinking-skills --force-with-lease

      - name: Open sync pull request
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh pr view sync/upstream-thinking-skills --json number >/dev/null 2>&1 || \
            gh pr create \
              --base main \
              --head sync/upstream-thinking-skills \
              --title "Sync upstream tjboudreaux/cc-thinking-skills" \
              --body-file - <<'BODY'
            Automated sync from `tjboudreaux/cc-thinking-skills`.

            Review vendored `skills/` changes before merge.
          BODY
```

- [ ] **Step 3: Validate workflow text**

Run:

```bash
python3 - <<'PY'
from pathlib import Path

path = Path(".github/workflows/sync-upstream.yml")
text = path.read_text()
for required in [
    "name: Sync upstream thinking skills",
    "https://github.com/tjboudreaux/cc-thinking-skills.git",
    "node scripts/validate-skills.js",
    "git push origin sync/upstream-thinking-skills --force-with-lease",
    "gh pr create",
]:
    if required not in text:
        raise SystemExit(f"missing required workflow text: {required}")
print("workflow text checks passed")
PY
```

Expected: `workflow text checks passed`.

- [ ] **Step 4: Commit**

Run:

```bash
git add .github/workflows/sync-upstream.yml
git commit -m "ci: add upstream sync workflow"
```

Expected: commit succeeds.

## Task 4: Smoke Test Plugin Shape and Skill Quality

**Files:**
- Read: `.claude-plugin/plugin.json`
- Read: `.codex-plugin/plugin.json`
- Read: `skills/*/SKILL.md`
- Read: `scripts/validate-skills.js`

- [ ] **Step 1: Run local validation checks**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json >/dev/null
python3 -m json.tool .codex-plugin/plugin.json >/dev/null
test -d skills
find skills -name SKILL.md -type f | grep -q .
node scripts/validate-skills.js
```

Expected: command exits 0.

- [ ] **Step 2: Inspect changed files**

Run:

```bash
git status --short
```

Expected: no uncommitted changes after previous task commits.

## Task 5: Update Plugin Marketplace

**Files:**
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`
- Modify if needed: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json`

- [ ] **Step 1: Update Claude marketplace source if needed**

In `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`, ensure the `thinking-skills` entry points at the maintained fork:

```json
{
  "name": "thinking-skills",
  "description": "39 mental models and frameworks for critical thinking in Claude Code",
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/agent-thinking-skills.git"
  }
}
```

If the fork is moved to an Accelerate Data-owned repository before
implementation, use that repository URL instead.

- [ ] **Step 2: Update Codex marketplace source**

In `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`, ensure the `thinking-skills` entry points at the same maintained fork:

```json
{
  "name": "thinking-skills",
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/agent-thinking-skills.git"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Productivity"
}
```

- [ ] **Step 3: Keep marketplace versions aligned**

If `.claude-plugin/marketplace.json` `metadata.version` changes, update
`/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json` to the same version.

- [ ] **Step 4: Validate marketplace**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
python3 scripts/validate-marketplace.py --base-ref HEAD
python3 -m unittest tests/test_validate_marketplace.py
codex plugin marketplace add .
```

Expected:

```text
marketplace validation passed
Ran 21 tests
OK
Marketplace `ad-internal-marketplace` is already added
```

- [ ] **Step 5: Commit marketplace update**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
git add .claude-plugin/marketplace.json .agents/plugins/marketplace.json .claude-plugin/plugin.json
git commit -m "feat: use maintained thinking skills fork"
```

Expected: commit succeeds.

## Self-Review

- Spec coverage: The plan covers Codex metadata, agent warning guidance,
  upstream sync workflow, local validation, and marketplace source updates.
- Placeholder scan: No placeholder markers or unspecified implementation steps remain.
- Scope check: The plan is one focused wrapper-and-marketplace task. Mental-model
  skill behavior changes are intentionally out of scope.
