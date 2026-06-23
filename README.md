# maestro-ai-github-workflows

Reusable GitHub Actions workflows central to Maestro AI. These workflows let you run a
**RuFlo hive-mind** — a queen-led team of AI agents — directly against any repository,
straight from the GitHub Actions tab. The agents read your code, make the changes you
asked for, and commit and push them back to the same branch automatically.

This repository ships two reusable workflows plus matching example callers:

| Reusable workflow | Provider | Example caller | Required secret |
| --- | --- | --- | --- |
| `.github/workflows/ruflo-hivemind.yml` | Anthropic (Claude) | `.github/workflows/ruflo-hivemind-example.yml` | `ANTHROPIC_API_KEY` |
| `.github/workflows/ruflo-openrouter.yml` | OpenRouter (DeepSeek, etc.) | `.github/workflows/ruflo-openrouter-example.yml` | `OPENROUTER_API_KEY` |

---

## Why run AI agent teams through workflows?

Running an agent team through a GitHub workflow removes nearly all of the setup labor
that normally stands between you and a working multi-agent run:

- **Zero local setup.** No Node.js install, no `npm install -g`, no RuFlo install, no
  Claude Code install, and no API keys sitting in your local shell. The runner builds a
  clean, reproducible environment every time.
- **One safe place for secrets.** Your API key lives once in GitHub Secrets, encrypted
  and never printed to logs. Nobody has to copy keys onto laptops or share them in chat.
- **Run from anywhere.** Kick off a run from the GitHub Actions tab (or the API) on any
  device — including a phone — without a development machine.
- **Reproducible and pinned.** RuFlo, the model, and the tool versions are all pinned in
  the workflow, so every teammate gets the same behavior instead of "works on my machine."
- **Hands-off results.** The agents do the work and the workflow commits and pushes the
  result back to your branch, so you review a normal diff/PR instead of babysitting a
  terminal.
- **Auditable.** Every run is logged, including the full agent output captured in the
  GitHub Step Summary, and every change lands as a reviewable commit.
- **Consistent and scalable.** The same workflow can be reused across many repositories
  and triggered consistently by your whole team.

---

## What the workflows do

Both workflows perform the same sequence of steps; they differ only in which AI provider
they authenticate against.

1. **Validate** that the required API key secret and the `instruction` input are present.
2. **Check out** the calling repository (full history, `fetch-depth: 0`).
3. **Set up Node.js 24** and install **Claude Code** (`@anthropic-ai/claude-code`).
4. **Configure headless Claude Code** so it runs non-interactively with permissions
   bypassed (`--permission-mode bypassPermissions`), suitable for CI.
5. **Initialize RuFlo** (`ruflo init`).
6. **Initialize the hive-mind** with a `hierarchical-mesh` topology and `byzantine`
   consensus (`ruflo hive-mind init`).
7. **Spawn the swarm** to carry out your `instruction` with a `strategic` queen
   (`ruflo hive-mind spawn "<instruction>"`).
8. **Commit and push** any changes back to the same branch as `github-actions[bot]`.
   The commit message is `ruflo: <first line of your instruction>`.

### What gets committed (and what does not)

To keep agent runs from polluting your repo with scaffolding, the commit step **excludes**:

- `.mcp.json` and `CLAUDE.md`
- Any changes under root-level dot directories (e.g. `.ruflo/`, `.claude/`, `.swarm/`),
  **except** `.github/`, which is allowed.

If the only changes are excluded paths (or there are no changes at all), the workflow
finishes cleanly without creating a commit.

---

## Prerequisites

Before calling either workflow you need:

1. **An API key secret** in the repository (or organization) that runs the workflow:
   - For the Anthropic workflow: `ANTHROPIC_API_KEY`
   - For the OpenRouter workflow: `OPENROUTER_API_KEY`

   Add it under **Settings → Secrets and variables → Actions → New repository secret**.

2. **`contents: write` permission**, granted by the caller job (the examples already do
   this). The default `GITHUB_TOKEN` must be able to push to the target branch — branch
   protection rules without a bypass will block the push.

---

## Using the example callers

The example workflows (`ruflo-hivemind-example.yml` and `ruflo-openrouter-example.yml`)
are ready-to-run `workflow_dispatch` callers. To use them:

1. Make sure the matching API key secret exists (see [Prerequisites](#prerequisites)).
2. Open the **Actions** tab in GitHub.
3. Select **RuFlo Hive Mind (example caller)** or
   **RuFlo Hive Mind OpenRouter (example caller)**.
4. Click **Run workflow**, fill in the inputs, and run it.
5. Watch the run, read the captured agent output in the **Step Summary**, and review the
   commit the workflow pushes to your branch.

### `ruflo-hivemind-example.yml` (Anthropic / Claude)

Required secret: **`ANTHROPIC_API_KEY`**.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `instruction` | Yes | `Review this repository and suggest one small documentation improvement.` | The task for the RuFlo agent team. Paths refer to the calling repo. |
| `model` | No | `claude-sonnet-4-6` | Anthropic model slug for the primary (Opus/Sonnet) tiers. |
| `fast_model` | No | `claude-haiku-4-5` | Anthropic model slug for the secondary (Haiku/subagent) tiers. |
| `ruflo_version` | No | `3.12.4` | Version of RuFlo to use. |

### `ruflo-openrouter-example.yml` (OpenRouter / DeepSeek)

Required secret: **`OPENROUTER_API_KEY`**.

This variant routes Claude Code through OpenRouter by pointing
`ANTHROPIC_BASE_URL` at `https://openrouter.ai/api` and using your OpenRouter key as the
auth token. That lets you drive the same RuFlo hive-mind with OpenRouter-hosted models
such as DeepSeek.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `instruction` | Yes | `Review this repository and suggest one small documentation improvement in file notes/improvement_suggestion.md` | The task for the RuFlo agent team. Paths refer to the calling repo. |
| `model` | No | `deepseek/deepseek-v4-pro` | OpenRouter model slug for the primary (Opus/Sonnet) tiers. |
| `fast_model` | No | `deepseek/deepseek-v4-flash` | OpenRouter model slug for the secondary (Haiku/subagent) tiers. |

---

## Calling the reusable workflows from another repository

You don't have to copy the examples — you can call the reusable workflows directly from
any repository. Create a workflow such as `.github/workflows/ruflo.yml`:

```yaml
name: RuFlo Hive Mind

on:
  workflow_dispatch:
    inputs:
      instruction:
        description: Task for the RuFlo agent team
        required: true
        type: string

jobs:
  ruflo:
    permissions:
      contents: write
    uses: AsperitasConsulting/maestro-ai-github-workflows/.github/workflows/ruflo-hivemind.yml@v1.0.0
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      instruction: ${{ inputs.instruction }}
      model: "claude-sonnet-4-6"
      fast_model: "claude-haiku-4-5"
```

For the OpenRouter variant, swap the `uses:` reference to
`ruflo-openrouter.yml`, pass `OPENROUTER_API_KEY` instead of `ANTHROPIC_API_KEY`, and use
OpenRouter model slugs:

```yaml
jobs:
  ruflo:
    permissions:
      contents: write
    uses: AsperitasConsulting/maestro-ai-github-workflows/.github/workflows/ruflo-openrouter.yml@v1.0.0
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
    with:
      instruction: ${{ inputs.instruction }}
      model: "deepseek/deepseek-v4-pro"
      fast_model: "deepseek/deepseek-v4-flash"
```

Pin the `@<ref>` (e.g. `@v1.0.0`) to a released tag so your runs stay reproducible.

---

## Tips and troubleshooting

- **Write clear instructions.** The `instruction` is the agents' whole brief. Be specific
  about what to change and which files/paths are in scope.
- **"No committable changes."** This means the run produced no changes outside the
  excluded paths. Refine your instruction or confirm the agents actually edited tracked
  files.
- **Push was rejected.** Check that the target branch isn't protected against
  `GITHUB_TOKEN` pushes, and that the caller job grants `contents: write`.
- **Authentication errors.** Confirm the correct secret name exists and is non-empty in
  the repository running the workflow.
- **Runs can take a while.** Each job allows up to 90 minutes; complex instructions on
  large repos use more of that budget.
