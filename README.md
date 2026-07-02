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
- `agents/skills/` for local and vendored role skills.
- `schemas/review-verdict.schema.json` for strict reviewer verdicts.
- `reports/agent-runs/_template/` for per-stage blackboard files.
- `docs/` as canonical approved project document slots.

## Install Into A Project

From this repository:

```bash
rsync -a \
  AGENTS.md CLAUDE.md agents docs reports schemas workflows \
  /path/to/your/project/
```

Then customize the target project's `agents/registry.yaml` adapter commands and
fill approved project documents only through the workflow.

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

This is a document-level Harness. It does not yet include an executable runner or
automatic model invocation.
