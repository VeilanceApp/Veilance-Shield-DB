# Veilance Shield Database

The **Veilance Shield Database** is the public, data-only rule source for [Veilance](https://veilance.org/) fingerprint protection.

Each JSON file describes:

- the browser behavior that should activate a shield;
- the packaged protection strategy Veilance should apply;
- tightly bounded parameters for that strategy; and
- the user-facing surface and explanation shown by Veilance.

Veilance downloads this repository automatically, validates every rule, and stores the last known good rule set locally. This lets protection rules be added, removed, enabled, or tuned without publishing a new extension version.

## Important boundary

Shield rules are **declarative data**, not remote JavaScript. A rule can only select a protection hook and strategy already implemented in the installed extension. This protects users and keeps the update system compatible with browser-extension store policies that prohibit remotely hosted code.

You can update the database without changing the extension when:

- enabling or disabling a supported protection;
- changing an existing rule's match details;
- changing bounded strategy parameters;
- adding another rule for an API/action hook the extension already exposes; or
- changing names, surfaces, and descriptions.

A new extension release is still required when adding a completely new browser API hook or a new protection strategy.

## Repository layout

```text
veilance-json-shields/       Individual production rules
schema/shield-rule.schema.json
scripts/validate-rules.mjs   Dependency-free repository validator
.github/workflows/validate.yml
```

The extension reads JSON files only from `veilance-json-shields/`. Files elsewhere in the repository are ignored by the downloader.

## Current rule pack

The initial pack contains 15 rules covering:

| Surface | Protected behavior |
|---|---|
| Canvas 2D | Pixel readback, data URL export, and blob export |
| WebGL | Unmasked vendor, unmasked renderer, and pixel readback |
| Web Audio | `getChannelData()` and `copyFromChannel()` readback |
| Navigator | Logical processor count, device memory, and touch points |
| Screen | Width, height, color depth, and device pixel ratio |

Protection is session-consistent: farbling uses a page-session seed, so repeated reads within the same page receive consistent treatment while a later page session uses a different seed. Normalization and bucketing reduce exposed entropy by returning common values.

## Rule format

One rule belongs in each `.json` file.

```json
{
  "schemaVersion": 1,
  "id": "navigator-hardware-concurrency",
  "name": "CPU thread count bucketing",
  "category": "Device characteristics",
  "description": "Rounds the reported logical processor count to a common even value and caps unusually high values.",
  "surface": "CPU characteristics",
  "defaultEnabled": true,
  "match": {
    "indicatorId": "navigator-characteristics",
    "api": "Navigator",
    "action": "read-hardware-concurrency"
  },
  "protection": {
    "strategy": "bucket-number",
    "parameters": {
      "step": 2,
      "minimum": 2,
      "maximum": 8,
      "rounding": "nearest",
      "preserveZero": false
    }
  }
}
```

### Top-level fields

| Field | Required | Meaning |
|---|---:|---|
| `schemaVersion` | Yes | Must currently be `1`. |
| `id` | Yes | Stable lowercase identifier. Use letters, numbers, dots, underscores, and hyphens. |
| `name` | Yes | Short human-readable name. |
| `category` | No | Group such as `Canvas`, `WebGL`, or `Device characteristics`. |
| `description` | Yes | Factual description of what changes. Do not label ordinary API use as malicious. |
| `surface` | Yes | Short label used in Shield activity. |
| `defaultEnabled` | No | Defaults to `true`; set to `false` to distribute a dormant rule. |
| `match` | Yes | Exact packaged hook to match. |
| `protection` | Yes | Allowlisted strategy and bounded parameters. |

### Match fields

`match` uses the same vocabulary as Veilance detection signals, but Shield matching is immediate. A shield must protect the first sensitive read, so frequency thresholds such as `minCount` are intentionally unsupported.

```json
{
  "match": {
    "indicatorId": "webgl",
    "api": "WebGL",
    "action": "renderer-query",
    "detail": {
      "parameter": "UNMASKED_RENDERER_WEBGL"
    }
  }
}
```

- `indicatorId`, `api`, and either `action` or `actions` are required.
- Use `actions` when one rule should cover closely related hooks.
- `detail` adds exact primitive-value matching. It is useful when one API/action exposes several parameters.
- Matches are case-sensitive after validation. Copy identifiers from an existing working rule or the extension hook documentation.

Example with multiple actions:

```json
{
  "match": {
    "indicatorId": "screen-characteristics",
    "api": "Screen",
    "actions": ["read-width", "read-avail-width"]
  }
}
```

## Supported protection strategies

Remote rules can use only the following strategies.

### `canvas-pixel-farbling`

Changes a small deterministic selection of RGB channel values in a protected canvas copy.

```json
{
  "strategy": "canvas-pixel-farbling",
  "parameters": { "maximumEdits": 8, "delta": 1 }
}
```

- `maximumEdits`: integer from 1 to 64.
- `delta`: integer from 1 to 8.

### `typed-array-farbling`

Changes selected integer values in a page-visible typed array, such as WebGL pixel output.

```json
{
  "strategy": "typed-array-farbling",
  "parameters": { "maximumEdits": 8, "delta": 1 }
}
```

### `float-array-farbling`

Changes selected floating-point samples by a very small amount.

```json
{
  "strategy": "float-array-farbling",
  "parameters": { "maximumEdits": 8, "epsilon": 0.0000001 }
}
```

- `epsilon`: from `0.000000001` through `0.001`.

### `bucket-number`

Rounds a number into a coarser bucket, then clamps it to a bounded range.

```json
{
  "strategy": "bucket-number",
  "parameters": {
    "step": 100,
    "minimum": 320,
    "maximum": 3840,
    "rounding": "nearest",
    "preserveZero": true
  }
}
```

`rounding` may be `nearest`, `floor`, or `ceil`.

### `replace-number`

Returns one common numeric value.

```json
{
  "strategy": "replace-number",
  "parameters": { "value": 24 }
}
```

### `binary-number`

Preserves whether a value is zero while hiding the exact positive value. This is useful for capability counts such as touch points.

```json
{
  "strategy": "binary-number",
  "parameters": { "zeroValue": 0, "nonZeroValue": 5 }
}
```

### `replace-string`

Returns a common string profile for a supported string-valued hook.

```json
{
  "strategy": "replace-string",
  "parameters": { "value": "Google Inc. (Google)" }
}
```

## Building a rule

1. Copy the closest rule in `veilance-json-shields/`.
2. Give it a unique, stable `id` and use that ID as the filename.
3. Select an `indicatorId`, `api`, and `action` already exposed by the extension.
4. Choose a compatible strategy from the allowlist above.
5. Start with conservative parameters. A shield that breaks normal site behavior is not useful protection.
6. Describe exactly what is changed and avoid claiming the underlying website behavior is malicious.
7. Run the validator.
8. Test both fingerprint resistance and normal behavior on representative sites before merging.

```bash
npm test
```

The validator rejects malformed JSON, unknown fields, duplicate IDs, filename/ID mismatches, unsupported strategies, and unsafe parameter ranges.

## Review checklist

Before merging a rule, verify:

- the rule validates with `npm test`;
- its ID is unique and will remain stable;
- its match corresponds to a packaged Shield hook;
- its strategy is compatible with the original return type;
- repeated reads are consistent within one page session;
- the change does not alter visible canvas output;
- common drawing, media, graphics, localization, and responsive-layout use cases still work;
- the description says what Veilance changes without overstating intent; and
- the rule does not include code, URLs to code, regular expressions intended as code, or unbounded values.

## Update and rollback behavior

The extension checks the repository every eight hours and on extension installation. It downloads the `main` branch archive, extracts only `veilance-json-shields/*.json`, and validates the complete candidate set before replacing the active set.

An update is rejected when it is empty, has too many validation failures, or unexpectedly removes more than half of an established rule set. If validation or download fails, Veilance keeps using the last known good rules and records the error in Settings.

For a bad database release, revert the offending commit on `main`. Clients will receive the restored archive on their next check. Do not reuse a removed rule ID for unrelated behavior.

## Compatibility and breakage risk

Fingerprint protection changes values websites receive, so zero compatibility risk is impossible. Prefer entropy reduction over aggressive spoofing, preserve broad capabilities where practical, and keep changes small. The initial pack is intentionally conservative, but WebGL, audio, screen, and device-profile changes can still affect specialized applications.

Veilance Shield remains opt-in. Detection and Shielding are independent: users can continue observing fingerprint attempts even when protection is disabled.
