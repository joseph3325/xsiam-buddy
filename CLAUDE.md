# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code **plugin** that helps security engineers develop Cortex XSIAM/XSOAR content. The plugin bundles skills (content generation workflows) and reference knowledge files. There is no runtime code — no build step, no tests, no package manager.

## Plugin Architecture

The plugin is installed via `claude plugin` and exposes skills to Claude Code. Skills are Markdown files with YAML front-matter (`SKILL.md`) that Claude reads to guide content generation. Each skill reads its own reference files before generating output.

```
.claude-plugin/
  plugin.json          # Plugin metadata (name, version, author)
  marketplace.json     # Marketplace manifest listing skills
skills/
  xsiam-scripts/       # Standalone Python scripts embedded in YAML
    SKILL.md
    references/script-yaml-spec.md
    references/script-types-patterns.md
  xsiam-integrations/  # Multi-command integrations with BaseClient
    SKILL.md
    references/integration-yaml-spec.md
    references/integration-patterns.md
  xsiam-event-collectors/ # Event collector integrations (data lake ingestion)
    SKILL.md
    references/event-collector-spec.md
  xsiam-xql/           # Standalone XQL query generation
    SKILL.md
    references/xql-examples.md
  xsiam-widgets/      # Dashboard widget queries with | view graph
    SKILL.md
    references/widget-view-graph-spec.md
  xsiam-correlations/  # Correlation rule JSON files with embedded XQL
    SKILL.md
    references/correlation-rule-spec.md
    references/correlation-examples.md
  xsiam-splunk-to-xql/ # SPL to XQL translation
    SKILL.md
    references/spl-to-xql-mapping.md
  xsiam-playbooks/     # Playbook YAML + companion docs
    SKILL.md
    references/playbook-format.md
  xsiam-docs-playbooks/ # Playbook documentation (HTML for Google Docs)
    SKILL.md
    references/playbook-doc-spec.md
    references/html-styling-guide.md
  xsiam-shared/
    references/
      common-patterns.md          # Shared Python patterns (scripts + integrations)
      xql-core-reference.md       # Shared XQL stages, functions, operators (always loaded)
      xql-datasets-core.md        # Shared XQL datasets, presets, joins (always loaded)
      xql-advanced-functions.md   # Advanced array/JSON/window functions (on-demand)
      xql-datasets-extended.md    # Third-party, email, CIE datasets (on-demand)
      xql-federated-search.md     # External S3/GCS/Azure querying (on-demand)
docs/plans/            # Design documents for significant changes
```

## How Skills Work

Each `SKILL.md` follows this pattern:
1. YAML front-matter with `name` and `description` (used for skill matching)
2. A "Before Starting" section listing which reference files to read first
3. A step-by-step workflow the model follows
4. A validation checklist before delivering output

Skills never generate code outside of YAML/JSON output files. All XSIAM content is delivered as unified `.yml` files (Python embedded inside YAML via `script: |-` for scripts, `script.script: |-` for integrations and event collectors), except correlation rules which are delivered as `.json` files matching the XSIAM export/import format.

The xsiam-xql, xsiam-correlations, and xsiam-splunk-to-xql skills use **tiered reference loading**. Core XQL references always load; specialized references (advanced functions, extended datasets, federated search) load on-demand based on query requirements. This reduces token cost for simple queries while keeping advanced knowledge available.

## Scripts vs Integrations vs Event Collectors

The key structural differences between the three Python skill types:

| | xsiam-scripts | xsiam-integrations | xsiam-event-collectors |
|---|---|---|---|
| Use when | Standalone data processing | Connecting to external APIs | Ingesting events into data lake |
| Python location in YAML | Top-level `script: |-` | Nested `script.script: |-` | Nested `script.script: |-` |
| Base class | None (direct demisto calls) | `BaseClient` subclass | `BaseClient` subclass |
| YAML fetch flag | N/A | `isfetch: true` | `isfetchevents: true` |
| Data output | `return_results()` | `demisto.incidents()` (fetch) | `send_events_to_xsiam()` |
| Data destination | War room / context | Incident queue | XSIAM data lake (XQL queryable) |
| Naming convention | Any | PascalCase | Must end with `EventCollector` |

## XQL Skill Family

Three skills share a common XQL reference layer under `xsiam-shared/references/`:

| Skill | Purpose | Output |
|---|---|---|
| xsiam-xql | Standalone queries (hunting, investigation, analytics) | XQL in code block with header comments |
| xsiam-correlations | Detection rules (correlation rule engineering) | Complete `.json` with embedded XQL |
| xsiam-splunk-to-xql | SPL translation (Splunk migration) | XQL in code block with "Translated from SPL" note |
| xsiam-widgets | Dashboard widget visualization queries | XQL with `| view graph` in code block with header comments |

## Key XSIAM Conventions to Preserve

- **YAML field order for scripts** matters — must match real XSIAM export order: `commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`
- **YAML field order for integrations** matters — must match real XSIAM export order: `commonfields → vcShouldKeepItemLegacyProdMachine → name → display → category → description → sectionorder → configuration → script`
- **`vcShouldKeepItemLegacyProdMachine: false`** — always present after `commonfields` in both scripts and integrations
- **`sectionorder`** — lowercase 'o' (not camelCase `sectionOrder`), simple list format matching real XSIAM exports
- **`supportedModules: []`** — must be the first field on every `configuration` parameter, every `commands` entry, and every `arguments` entry in integration/event-collector YAML. For scripts, every `args` entry must have it as its first key.
- **Never include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- **Always include** `register_module_line()` calls as the first and last lines of embedded Python — required for platform line-number tracking
- Docker image: always pinned `3.12.x` (e.g., `demisto/python3:3.12.12.6947692`), never `:latest`
- `demisto.alert()` for XSIAM; `demisto.incident()` is XSOAR equivalent
- Arg defaults use `defaultValue:` (not `default:`); list args include `isArray: true`
- Config params should include `additionalinfo:` tooltips and `section:` for tab assignment

## When Editing Skills or References

- Skills reference their own reference files with relative paths (e.g., `references/script-yaml-spec.md`, `../xsiam-shared/references/common-patterns.md`)
- The `xsiam-shared/references/` directory contains both shared Python patterns (`common-patterns.md`, used by scripts and integrations) and shared XQL references (`xql-core-reference.md`, `xql-datasets-core.md`, etc., used by xsiam-xql, xsiam-correlations, and xsiam-splunk-to-xql) — changes to shared files affect all consuming skills
- `marketplace.json` lists the skills exposed by the plugin; add new skills there when creating a new skill directory
- Design documents for significant changes go in `docs/plans/` with filename format `YYYY-MM-DD-feature-name.md`

## Publishing

Install locally for testing:
```bash
claude plugin install xsiam-buddy@xsiam-buddy
```

Publish to marketplace:
```bash
claude plugin marketplace add joseph3325/xsiam-buddy
```

Version is tracked in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — update both when bumping.
