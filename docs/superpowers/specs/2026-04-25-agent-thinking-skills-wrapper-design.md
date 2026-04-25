# Agent Thinking Skills Wrapper Design

## Context

`agent-thinking-skills` is a fork of `tjboudreaux/cc-thinking-skills`, a
collection of 39 mental-model skills. The upstream repository already has a
Claude plugin manifest and a root `skills/` directory, but it does not have a
Codex plugin manifest.

Review of the upstream skill files found that the runtime skill content is
portable. Claude-specific references appear mainly in repository documentation
and development tooling, not in the `skills/*/SKILL.md` instructions. The skills
do not depend on Claude-only tools, slash commands, `.claude/` paths, or
scripts.

## Goals

- Make the thinking skills collection installable through both Claude and Codex
  marketplace entries.
- Keep the upstream `skills/` directory as real root files.
- Add a Codex plugin manifest while preserving the existing Claude plugin
  manifest.
- Make the repository's forked/upstream-owned nature explicit to AI agents.
- Require agents to warn before editing upstream-owned skill content.
- Keep future upstream syncs reviewable.

## Non-Goals

- Rewrite the mental-model skill content for Codex.
- Rename the individual thinking skills.
- Change upstream quality scripts.
- Automatically merge upstream changes without a pull request review.

## Repository Shape

The fork keeps the upstream layout intact:

```text
agent-thinking-skills/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  AGENTS.md
  README.md
  skills/
    thinking-archetypes/
    thinking-bayesian/
    ...
    thinking-via-negativa/
  scripts/
  .github/workflows/sync-upstream.yml
```

The `skills/` directory stays at the repository root as ordinary files. No
symlink or generated-copy layer is needed because upstream already has the
runtime shape expected by plugin installers.

## Marketplace Identity

Use `thinking-skills` as the marketplace-facing plugin name to preserve the
existing marketplace identity. The source URL should point at the maintained
fork after `.codex-plugin/plugin.json` exists:

```json
{
  "source": "url",
  "url": "https://github.com/accelerate-data/agent-thinking-skills.git"
}
```

If the repository is moved into the Accelerate Data organization, the source URL
should be changed to the org-owned URL before updating the marketplace.

## Agent Safety Note

`AGENTS.md` should state that this repository is a maintained fork of
`tjboudreaux/cc-thinking-skills`. AI agents may edit wrapper-owned files such as
`.codex-plugin/`, `.github/workflows/`, documentation, and validation scripts as
normal.

Before editing files under `skills/`, agents must warn the user that the file is
upstream-owned and explain that the change may diverge from future upstream
syncs. They may proceed only when the user explicitly confirms the local
divergence or when the task is specifically to resolve an upstream sync conflict.

## Upstream Sync

A scheduled GitHub Actions workflow should fetch `tjboudreaux/cc-thinking-skills`,
merge the upstream default branch into a sync branch, run validation, and open a
pull request. The workflow should not push directly to `main`.

The sync PR gives the team a review point for upstream skill changes and for any
conflicts against local wrapper files.

## Validation

The implementation should verify:

- `.claude-plugin/plugin.json` is valid JSON.
- `.codex-plugin/plugin.json` is valid JSON.
- `skills/` exists and contains `SKILL.md` files.
- Skill validation still passes with `node scripts/validate-skills.js`.
- GitHub Actions workflow text includes the upstream repository and pull request
  creation flow.
- The plugin can be referenced from `plugin-marketplace` as a `url` source.

Marketplace changes belong in `plugin-marketplace`, not in this fork.
