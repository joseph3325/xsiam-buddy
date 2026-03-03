# ExtractDomain — Generation Summary

## What Was Generated

A single unified YAML file (`ExtractDomain.yml`) ready for direct import into a Cortex XSIAM or XSOAR tenant. The file embeds the Python automation code inside a top-level `script: |-` field, following the XSIAM unified-YAML format.

## What the Script Does

`ExtractDomain` is a **transformer** script. It accepts a URL string via the `value` argument and returns the domain portion of that URL. For example:

- `https://www.example.com/path?q=1` → `www.example.com`
- `http://sub.domain.org:8080/page` → `sub.domain.org`
- `www.example.com/no-scheme` → `www.example.com`

## Key Decisions

### `value` as the argument name
The spec states that `transformer` and `filter` scripts receive their input via a `value` argument by convention. Because this script was described as a transformer, I used `value` as the single required argument and also marked it `default: true` so it can be passed without a named argument in the war room.

### Tag: `transformer`
The task description explicitly calls this a "transformer script". The `transformer` functional tag wires the script into the playbook task input-mapping UI so users can select it as a transformer on any field. I also added the informational `Utility` tag for discoverability.

### URL parsing approach
`urllib.parse.urlparse` (standard library, no extra dependencies) is used to split the netloc. If the input URL has no scheme, `https://` is prepended so `urlparse` correctly identifies the netloc rather than treating it as a path. Port numbers (e.g., `:8080`) are stripped from the result so only the bare hostname is returned.

### No `demisto.alert()` usage
This script does not operate on alert context — it is a pure data-transformation utility. Therefore `demisto.alert()` is not called.

### No platform-injected or content-pack-only fields
The following were intentionally omitted per the spec: `fromversion`, `marketplaces`, `tests`, `timeout`, `register_module_line` guards (they are bare calls at module scope, not wrapped in `if __name__`).

### Docker image
Pinned to `demisto/python3:3.12.12.6947692` (standard Python 3.12 image as specified in the reference). `urllib.parse` is part of the standard library so no additional packages are required.

## Validation Checklist (from SKILL.md)

- [x] Python code embedded via top-level `script: |-`
- [x] Python indentation consistent (2-space relative to key level inside YAML block)
- [x] Standard imports present (`demistomock`, `CommonServerPython`, `CommonServerUserPython`)
- [x] `main()` has `try/except` with `return_error()`
- [x] Field ordering matches spec: `commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`
- [x] `vcShouldKeepItemLegacyProdMachine: false` present
- [x] `enabled: true`, `scripttarget: 0`, `pswd: ""`, `runas: DBotWeakRole`, `engineinfo: {}`, `mainengineinfo: {}` all present
- [x] Every `args` entry has `supportedModules: []` as first key
- [x] No `defaultValue` needed (arg is required); `isArray` omitted (single value)
- [x] All args have descriptions
- [x] All outputs have `contextPath`, `description`, and `type`
- [x] Docker image is pinned `3.12.x` version
- [x] `fromversion`, `marketplaces`, `tests`, `timeout` not included
- [x] First line of embedded Python is `register_module_line('ExtractDomain', 'start', __line__())`
- [x] Last line of embedded Python is `register_module_line('ExtractDomain', 'end', __line__())`
- [x] No tab characters; consistent YAML indentation throughout
- [x] `demisto.alert()` not called (not an alert-context script)
- [x] Arg parsing uses `args.get()` — no raw casting (string input requires no numeric/boolean conversion)
