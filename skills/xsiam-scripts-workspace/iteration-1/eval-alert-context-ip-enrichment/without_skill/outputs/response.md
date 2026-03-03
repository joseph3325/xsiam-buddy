# EnrichAlertIPs — Generation Summary

## What Was Generated

A single unified YAML file (`EnrichAlertIPs.yml`) containing an XSIAM automation script ready for direct tenant import. The script reads `src_ip` and `dest_ip` from the current alert context, calls the platform `ip` enrichment command on each address, and returns a consolidated summary.

## Decisions Made

### Alert Context Access
Used `demisto.alert()` (not `demisto.incident()`) to read the current alert's fields, as required for XSIAM alert-context automations. The script gracefully handles the case where either field is absent or empty — it logs a debug message and skips that address rather than failing.

### IP Enrichment Approach
Used `demisto.executeCommand('ip', {'ip': ip})` with a manual `isError()` check rather than `execute_command()`. This gives finer control: a failure on one IP address (e.g., no enrichment integration configured) produces an error message in the summary row rather than aborting the entire script. This seemed more appropriate for a "best-effort enrichment" use-case where src and dest may be independently enrichable.

### Structured Context Extraction
The `extract_enrichment_summary()` helper parses the raw entry context returned by the `ip` command to surface the four most actionable fields: DBotScore verdict + vendor, ASN, and country. These are written to `EnrichAlertIPs.*` context paths for downstream playbook use.

### Raw Entry Passthrough
After building the summary, raw non-error entries from the `ip` command are passed through via `demisto.results()`. This ensures the platform's standard entry-context machinery also writes `DBotScore`, `IP`, and other indicator context keys into the investigation — so any subsequent playbook tasks that look for `DBotScore(val.Indicator == "x.x.x.x")` will find them.

### No Script Arguments
The script takes no arguments — all input comes from `demisto.alert()`. This matches the stated task (read from the current alert) and avoids requiring the caller to pass IPs manually.

### YAML Conventions Followed
- Field order matches real XSIAM export order (`commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`)
- `register_module_line()` calls wrap the embedded Python as first/last lines
- Docker image pinned to `demisto/python3:3.12.12.6947692`
- No `fromversion`, `marketplaces`, `tests`, or `timeout` fields included
- `args: []` declared explicitly since the script takes no arguments
- `runas: DBotWeakRole` as required

## Output File

`EnrichAlertIPs.yml` — import via **Settings → Automations → Import** in XSIAM, or reference by name in a playbook task.
