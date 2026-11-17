# Tool output contract

All commands emit JSON to stdout. Fatal usage or index errors emit a JSON object to stderr and return a non-zero exit code.

## Runtime query version selection

`search`, `get`, `members`, `inheritance`, and `resolve-location` include `versionSelection: { requestedVersion, apiVersion, fallback }`, including `NOT_FOUND` responses. Exact versions win; otherwise select the newest cached version no higher than the requested version. If every cached version is newer, select the oldest available version. Ordering compares numeric major/minor/patch and prerelease identifiers; build metadata does not change precedence. Returned symbols and evidence retain the actual selected version. Unparseable missing versions fail instead of guessing. This selection does not bypass database readability or file integrity checks and does not apply to `compare-versions`, `status`, or `verify-index`.

`requestedVersion` is the target being answered for; `apiVersion` is the declaration snapshot label. Under the knowledge base's maintained compatibility policy, the automatically selected snapshot supplies compatible API evidence for the target. `fallback: true` only records that the labels differ; it is not an incompatibility or unverified-compatibility status. For example, `{ requestedVersion: "3.4.2", apiVersion: "3.4.0", fallback: true }` supplies compatible API evidence for 3.4.2 from the 3.4.0 snapshot. Keep the actual evidence labels intact and omit routine version-selection warnings from user-facing answers.

## Commands

### `cache-update`

Checks or force-checks the remote cache manifest and returns `success`, `action`, `usableCache`, and `cachePath`. `action` is one of `installed`, `not-modified`, `up-to-date`, `throttled`, `retry-deferred`, `locked`, `unconfigured`, or `failed`. A failed remote update with `usableCache: true` is non-fatal: continue using the active cache. `--force` bypasses TTL and retry deferral but never bypasses file integrity or database readability validation. It does not compare API versions, declaration hashes, generator versions, or minimum Skill versions against bundled metadata.

### `versions`

Returns the declaration versions present in the current generator source or downloaded database, ordered newest first.

### `status`

Inspects one selected version and reports `missing`, `ready`, `outdated`, or `corrupt` without building an index. Returns the shared Skill-relative index path, selected-version counts, generation time, and `indexedVersions`.

### `verify-index`

The published runtime checks SQLite integrity, selected API version, readable Schema, and symbol counts without rebuilding or comparing bundled declaration hashes. Returns `valid`, state, and individual checks.

### `manifest`

In the private generator CLI, returns the selected API version, declaration-file list, source domains, byte sizes, SHA-256 values, and deterministic Manifest hash. This build-time command is not required by the published query runtime.

### `index`

Builds or reuses the local SQLite-compatible incremental index and returns:

- `indexPath`
- `apiVersion`
- `manifestSha256`
- `created`
- `action` (`reused`, `added`, `updated`, or `rebuilt`)
- `indexedVersions`
- `symbolCount`
- `parameterCount`
- `relationCount`

### `search <query>`

Returns bounded candidates with:

- version-specific content-revision `symbolId`
- `fullyQualifiedName`
- symbol `kind`
- normalized `signature`
- documentation summary
- `deprecated`
- `matchReason`
- evidence

Add `--owner <FQN>` to restrict results to direct members of one symbol. Recognized Chinese domain concepts may return `matchReason: "alias"`.

### `get <symbol>`

Accepts a stable ID, exact FQN, or exact name. Returns the complete symbol, parameters, relations, documentation, deprecation state, and evidence.

### `members <symbol>`

Returns direct members. Add `--inherited` to include resolved base classes and interfaces. Each member identifies `declaredBy` and `inheritedDepth`.

### `inheritance <symbol>`

Returns the root symbol plus bounded `nodes` and `edges`. Use `--direction ancestors|descendants`, `--max-depth N`, and `--limit N`. The response reports `truncated` and `cycleDetected`; every node includes normal evidence fields.

### `resolve-location <symbol>`

Resolves an indexed symbol to declaration evidence. Returns only `skillRelativePath`, line range, version, source domain, and source hash; it never returns an absolute machine path. The published runtime does not ship the private declaration source files, so callers should use the returned project-relative declaration evidence when available.

### `compare-versions [symbol]`

Compares two bundled, indexed declaration versions selected by `--from-version` and `--to-version`. When both options are omitted, the newest two bundled versions are used. Passing only one is invalid.

The response includes `summary` counts and bounded `changes`. Each change is `added`, `removed`, or `modified` and carries the stable `identityId` plus version-specific `from` and `to` symbol snapshots. Modified entries list `changedFields` from `signature`, `documentation`, and `deprecated`. `--diff-type class|method|property|all`, `--domain`, an optional symbol/owner name, and `--limit` narrow the comparison. The optional name includes the matching type and its direct members, so class upgrade checks can detect member changes without scanning unrelated output.

`totalChanges` is calculated before limiting and `truncated` reports omitted changes. Unchanged overload identities are counted in `summary.unchanged` and are not emitted as changes.

### `schema-search [query]`

Searches `data/json/component_schemas.json` by node/component name or description. `--world-type 2d|3d` filters by the effective world type declared by the schema or inherited from its base schemas. Results contain `name`, direct `base`, effective `worldType`, `description`, and direct `propertyCount`. `total` is calculated before `--limit` is applied.

### `schema-get <name>`

Returns the exact node/component JSON Schema. Matching is case-insensitive. Add `--inherited` to return:

- `inheritanceChain`, ordered from the direct parent outward
- `inheritedSchemas`, preserving each parent schema
- `effectiveProperties`, flattened from the oldest ancestor to the selected type, with `declaredBy`

A missing exact name returns `NOT_FOUND` with bounded spelling and partial-name suggestions.

### `schema-descendants <name>`

Returns all derived node/component schemas using the local `$ref` inheritance graph. Use `--max-depth 1..20`, `--limit N`, and optionally `--include-properties`. The response includes each subclass depth, `subclassCount`, and `truncated`.

## Evidence

Every symbol result contains:

- `schemaVersion`
- `apiVersion`
- `manifestSha256`
- `sourceDomain`
- `sourcePath`
- `sourceSha256`
- `startLine`
- `endLine`

Use these fields to distinguish engine versions and runtime environments.

Component schema results use separate dataset evidence: `sourcePath`, `sourceSha256`, and `schemaCount`. The current JSON dataset is shared and unversioned, so schema commands reject `--api-version` rather than returning misleading version evidence.
