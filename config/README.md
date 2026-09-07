# Usage parser configuration (schema 1)

CodeAgentGauge downloads `config/usage-parser-v1.json` from this public repository.
The app checks once per day by default while running. Settings offers 1/6/12 hours,
1/3/7 days, Off, and **Check Now**, which bypasses the app's interval and HTTP cache.
Overdue checks run at launch / after wake. This is a **data mapping update**, not an
app binary update. Users need the app version that includes this updater first.

## Publishing a mapping change

1. Edit `config/usage-parser-v1.json` on the `main` branch.
2. Increment the integer `revision` (1 → 2 → 3…). Keep `schemaVersion: 1`.
3. Preserve every existing rule and output field. Change its input `paths` instead.
4. Validate and test with representative **synthetic** payloads; never commit tokens,
   auth files, account identifiers, private logs, or captures.
5. Commit/push. In the app, use Settings → Usage Parser Configuration → Check Now.
   Confirm the displayed revision and actual usage. A routine daily check will also apply it.

The app ships the same baseline file at
`Sources/CodexUsageMonitor/Resources/usage-parser-v1.json` in
[h0tshin/codex-usage-monitor](https://github.com/h0tshin/codex-usage-monitor).
When shipping a new binary, copy the currently validated configuration into that
resource. The bundled revision is the minimum accepted revision.

## Mapping contract

`rules` converts incoming JSON to a stable internal JSON shape. All paths are arrays
of literal key names (or array indices), relative to the current object, not JSONPath,
scripts, filesystem paths, or URLs. Paths are tried in order; absent and null values
try the next path. Missing fields stay missing, never fabricated as zero.

Example: if the server moves the main limit from `rate_limit` to `data.quota`:

```json
"rate_limit": {
  "paths": [["data", "quota"], ["rate_limit"]]
}
```

For existing and new formats, keep the old path as a fallback. The app's fixed object
graph recursively normalizes nested limits, windows, model arrays, coupons and logs.

Optional numeric conversion supports a source number or numeric string:
`output = input * numberScale + numberOffset` (defaults are 1 and 0).

```json
"used_percent": {
  "paths": [["remaining_percent"]],
  "numberScale": -1,
  "numberOffset": 100
}
```

For minutes → seconds, set `numberScale: 60`. For milliseconds → seconds, set
`numberScale: 0.001`. Output must still satisfy the app's typed data contract.

For renamed event values, `stringValues` maps literal strings:

```json
"type": {
  "paths": [["kind"], ["type"]],
  "stringValues": {"usage_event": "event_msg"}
}
```

If log discriminator/field names change, also update `logMarkers` to include literal
strings present in those JSONL records. Markers prefilter bounded log reads; retain
old markers for older logs. Normalization still checks the event type after filtering.

| Rule | Input / stable output |
| --- | --- |
| accountUsage | Account usage response: main quota, extra models, plan, reset-credit count |
| accountLimit / accountWindow | Primary/secondary windows; duration in seconds and Unix reset seconds |
| additionalLimit | Model array element: server-issued code, label, windows |
| creditsBalance / creditsCount | Balance metadata / available reset-credit count |
| resetCredits / resetCredit | Coupon list/count and each coupon (ISO-8601 date strings) |
| sessionEvent / sessionPayload | JSONL envelope, timestamp, type, session identity, model, info, rate limits |
| tokenInfo / tokenAmount | Cumulative/latest token counts (total, input, output) |
| localLimit / localWindow | Local log quota; duration in minutes and Unix reset seconds |

Root paths can also include an empty array `[]` to select the current object when an
upstream wrapper has been removed. Authentication, endpoints, trusted quota identity,
aggregation algorithms, permissions, and application code are deliberately **not**
remotely configurable. Entirely new protocols, XML input, new output types, timestamp
encodings beyond scalar conversion, or new aggregation semantics need an app update.

## Safety and recovery

- Anonymous HTTPS GET to one fixed raw.githubusercontent.com URL; no credentials,
  user usage, file paths, or captures are uploaded to GitHub.
- No redirects. Maximum download size: 128 KiB; request/resource timeouts: 20/30 seconds.
- Schema/field/path/transform bounds validated before atomic persistence and activation.
- Older revisions, reused revisions with changed contents, and malformed files are rejected.
- Network or validation/write failures keep the previous configuration and show a failure
  message; the next scheduled attempt follows the chosen interval (no retry storm).
- An update clears incremental log parsing and in-memory coupon caches on the next read.
- To roll back a bad mapping, publish the previous mappings with a **higher revision**.
- Successfully downloading a structurally valid mapping does not prove its interpretation
  is correct: test real-shaped sanitized fixtures before publication. Current account
  lookup failures retain the app's existing offline fallback behavior.

## Developer checks

In the app repository:

```sh
swift test --filter 'UsageParsingConfigurationTests|UsageParserUpdateTests|ChatGPTUsageServiceTests|LocalCodexTokenUsageScannerTests'
VERIFY_PUBLISHED_PARSER=1 swift test --filter UsageParserUpdateTests/testLivePublishedGitHubConfigurationWhenRequested
```

The second command checks the live GitHub file against the bundled baseline; after a
configuration-only hotfix, update that baseline before expecting equality.
