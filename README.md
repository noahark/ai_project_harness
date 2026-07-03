# AI Project Harness

Reusable multi-model project Harness for requirements discussion, direction
drafting, stage design, implementation, review, fix, and evidence tracking.

The repository is a template only. It intentionally contains no product-specific
code, requirements, architecture, or business decisions.

## What This Provides

- `AGENTS.md` as the top-level agent authority file.
- `workflows/templates/feature-loop.yaml` as the stage workflow contract.
- `agents/registry.yaml` for model adapters, provider identity, and skill
  routing.
- `docs/model-adapters.md` for local CLI commands, model defaults, permission
  modes, and adapter availability checks.
- `agents/skills/` for local and vendored role skills.
- `schemas/review-verdict.schema.json` for strict reviewer verdicts.
- `reports/agent-runs/_template/` for per-stage blackboard files.
- `docs/` as canonical approved project document slots.
- `harness-manifest.yaml` as the source of truth for sync ownership.
- `scripts/install-harness.sh` and `scripts/update-project-harness.sh` for
  first install and later upgrades.

## Install Into A Project

From this repository:

```bash
scripts/install-harness.sh /path/to/your/project
```

The install script copies both `harness_owned` and `install_only` manifest
entries. Existing files are skipped unless `--force` is passed.

## Update An Existing Project

From this repository:

```bash
scripts/update-project-harness.sh /path/to/your/project
```

The update script copies only `harness_owned` manifest entries. It does not
overwrite project-owned PRD, architecture, roadmap, decision log, or project
README files. Dirty target git worktrees are rejected unless `--allow-dirty` is
passed.

When running an installed project-local copy of the script, pass the template
source explicitly:

```bash
scripts/update-project-harness.sh \
  --source /path/to/ai_project_harness \
  /path/to/your/project
```

Each install or update writes `.harness-version` in the target project.

## Canonical Project Paths

Drafts, model outputs, reviews, and test logs belong in:

```text
reports/agent-runs/<stage-id>/
```

Approved project documents belong in:

```text
docs/product/PRD.md
docs/architecture/ARCHITECTURE.md
docs/architecture/ADR/
docs/development/DEVELOPMENT_GUIDE.md
docs/planning/ROADMAP.md
docs/planning/DECISIONS.md
```

## Validate

```bash
ruby -e 'require "yaml"; %w[agents/registry.yaml workflows/templates/feature-loop.yaml].each { |f| YAML.load_file(f); puts "YAML OK #{f}" }'
ruby -e 'require "json"; %w[reports/agent-runs/_template/status.json schemas/review-verdict.schema.json].each { |f| JSON.parse(File.read(f)); puts "JSON OK #{f}" }'
git diff --check
```

## Current Scope

This is a document-level Harness plus executable validation gates. It records
model adapter command templates, but it still does not include a full automatic
multi-model runner.
