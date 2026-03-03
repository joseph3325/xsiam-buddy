# Script Generation Summary

## What Was Generated

**File:** `enrich-alert-ips.yml`

A unified XSIAM script YAML ready for direct import into a Cortex XSIAM tenant.

## Script Behaviour

1. Calls `demisto.alert()` to retrieve the current alert object and immediately flattens `CustomFields` into the top-level dict (required XSIAM normalization pattern).
2. Reads `src_ip` and `dest_ip` from the flattened alert.
3. For each IP that is present, calls `execute_command('ip', {'ip': ip})` to invoke whatever IP enrichment integration is active in the tenant.
4. Handles enrichment failures per-IP with a `try/except` so that a failure on one IP does not prevent the other from being enriched.
5. Builds a flat summary row per IP (IP address, role, ASN, organization, country, vendor, score) and renders it as a Markdown table via `tableToMarkdown()`.
6. Any per-IP enrichment errors are appended to the readable output as a bullet list.
7. Returns a `CommandResults` object whose `outputs` writes the full enrichment data to `IPEnrichmentSummary.Result` in the incident context.

## Key Decisions

- **No script arguments**: The task said to read from the alert context directly. No args are needed; the script auto-discovers `src_ip` / `dest_ip` from `demisto.alert()`.
- **Graceful handling of missing IPs**: If both `src_ip` and `dest_ip` are absent, the script returns a human-readable message rather than raising an error.
- **Per-IP error isolation**: Each enrichment call is wrapped individually so a failure for one IP does not abort enrichment of the other.
- **`execute_command()` preferred over `demisto.executeCommand()`**: As specified in the skill, `execute_command()` is the CommonServerPython alias that raises automatically on error.
- **`args: []`**: YAML list kept explicit (not omitted) so the import is unambiguous. No args are exposed because all inputs come from alert context.
- **Output type is `array`**: `IPEnrichmentSummary.Result` is a list of objects, one per IP enriched.
- **Script name `enrich-alert-ips`**: kebab-case as required by the skill conventions; used exactly in both `register_module_line` calls and the `name` / `commonfields.id` fields.

## Validation Checklist (Passed)

- [x] Python embedded via `script: |-` (top-level string)
- [x] Python indentation consistent (2-space)
- [x] Standard imports: `demistomock`, `CommonServerPython`, `CommonServerUserPython`
- [x] `main()` has `try/except` with `return_error()`
- [x] Field order matches spec: `commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`
- [x] `vcShouldKeepItemLegacyProdMachine: false` present
- [x] `enabled: true`, `scripttarget: 0`, `pswd: ""`, `runas: DBotWeakRole`, `engineinfo: {}`, `mainengineinfo: {}` all present
- [x] No args entries, so `supportedModules` constraint is not applicable (empty list)
- [x] No `fromversion`, `marketplaces`, `tests`, or `timeout` fields
- [x] First line of Python: `register_module_line('enrich-alert-ips', 'start', __line__())`
- [x] Last line of Python: `register_module_line('enrich-alert-ips', 'end', __line__())`
- [x] Docker image pinned to `demisto/python3:3.12.12.6947692`
- [x] Uses `demisto.alert()` (not `demisto.incident()`) for XSIAM alert context
- [x] `CustomFields` immediately flattened after `demisto.alert()` call
- [x] All outputs have `contextPath`, `description`, and `type`
- [x] No tab characters; consistent YAML indentation
