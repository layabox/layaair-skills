# Query playbook

## Reuse a query process

When the relevant MCP tool is unavailable, you can reuse the Skill script process for related queries.

1. In a Node.js session that can own a child process for the lookup, call `fork(skillScript, ["query-process"])` from `node:child_process`. Set `skillScript` to the absolute path of `scripts/layaair-api.cjs` in this Skill. Add `--project <path>` or `--cache-root <path>` to the startup arguments when needed.
2. Send `{ id: 1, args: ["get", "Laya.Sprite", "--api-version", "3.4.0"] }` over IPC. Give each outstanding request a distinct numeric ID and match replies by ID. Parse the `json` string on success and handle the `error` field on failure. Keep the same child for related queries. Run `status`, `verify-index`, and `cache-update` as separate CLI invocations.
3. After the last reply, call `child.disconnect()` to close the session and let the child exit. Fall back to the CLI if the execution tool cannot preserve the child and its IPC channel between lookups.

## Choose a domain

Use `engine-runtime` for `Laya`, scene nodes, components, rendering, physics, animation, audio, input, networking, and other game runtime code.

Use `ide` for `IEditor`, `IEditorEnv`, `IEditorClient`, `gui`, Blueprint, editor windows, asset database operations, scene management, build tools, and plugin UI code.

Use no domain filter only when the user explicitly asks for a cross-domain search. Explain domain differences in the result.

## Resolve ambiguity

1. For game UI tasks, first read current `modules` and `addons` via `ProjectManagement.getProjectInfo` (or project `settings/PlayerSettings.json` without MCP). Choose the configured UI family before searching; `addons["laya.ui"] = "ui2"` selects new UI types such as `Laya.GImage`.
2. Search the short name.
3. Compare `fullyQualifiedName`, `kind`, and `sourceDomain`.
4. Select a fully qualified symbol.
5. Run `get` with that FQN.
6. Ask the user only when multiple candidates remain valid for the requested environment.

## Handle overloads

Search results may contain multiple entries with the same FQN and different signatures. Preserve all applicable overloads. Select an overload from its parameter count and types; do not merge signatures.

## Handle inherited members

Run `members <FQN> --inherited` when a requested member is not declared directly on the class or interface. Use `declaredBy` and `inheritedDepth` to explain its origin.

Run `inheritance <FQN> --direction ancestors` to explain a complete base path. Run it with `--direction descendants` to discover derived classes and interfaces. Bound broad descendant queries with `--max-depth` and `--limit`.

## Search concepts and owners

Use `--owner <FQN>` when the user asks for a concept within a known class. The search command maps common Chinese LayaAir concepts to deterministic member-name aliases and marks those results with `matchReason: "alias"`. Treat aliases as candidate recall and still run `get` before using a signature.

## Verify generated code

Before emitting LayaAir code:

1. Verify every unfamiliar LayaAir class and called member.
2. Confirm required parameters, optional parameters, and return type.
3. Check `deprecated`.
4. Keep runtime and IDE domains separate.
5. Pass the requested target with `--api-version`, or use the current project's detected version. Use the automatically selected compatible snapshot for that target. `versionSelection.fallback` records snapshot selection, not a compatibility warning. Type-check generated code against the project declarations as normal code validation.

## Inspect referenced types

Use `get`, `members`, and `inheritance` to inspect LayaAir types named in API signatures. If a referenced type cannot be found, report it as unresolved rather than inferring an API from a similar name.

## Compare API versions

Run `compare-versions --from-version <old> --to-version <new>` for an engine upgrade. Add a class/interface FQN to focus on that declaration and its direct members. Use `--diff-type method` or `property` when only one compatibility surface matters.

- Treat `removed` APIs and incompatible `signature` changes as migration work.
- Treat `added` overloads separately from existing overloads; stable overload identities prevent unchanged signatures from being reported as modifications.
- Inspect both `from.evidence` and `to.evidence` before describing a change.
- A documentation-only or deprecation-only modification is evidence from the declarations, not necessarily a runtime behavior change.

## Query serialized node and component properties

Do not infer scene or prefab serialization fields from a TypeScript class declaration. Search with `schema-search`, select an exact type, then run `schema-get <name> --inherited`. Use `effectiveProperties` for the complete allowed property set and preserve each property's `type`, `default`, `range`, `enumSource`, `inspector`, `options`, and `declaredBy` metadata when relevant.

Use `schema-descendants` when choosing a concrete type derived from `Node`, `Sprite`, `Sprite3D`, `Component`, or another base. These commands inspect a shared schema dataset rather than the versioned declaration index; never attach an API version to their evidence.

## Handle failures

- `NOT_FOUND`: report that the selected version does not contain the symbol, then search a shorter concept or related type.
- New version: add it only when it is newer than the latest indexed version; unchanged API revisions are reused.
- Manifest change: allow correction of the latest indexed version only. Older or intermediate versions are intentionally rejected.
- Missing exact version: ordinary runtime API queries automatically select a compatible snapshot under the policy in `SKILL.md` and expose `versionSelection`. This is normal query behavior, not a failure or a reason to describe target compatibility as unverified. Comparisons and diagnostics still require exact versions; use `versions` to inspect the available data.
- Missing project: pass `--cache-root <path>` for the shared IDE cache, or run inside a LayaAir project / pass `--project <path>` for a project-local cache.
- Index build failure: report the tool error and declaration path; do not answer the API fact from memory.
- Missing component schema: use returned suggestions or `schema-search`; do not substitute a similarly named TypeScript class without schema evidence.

Run `status --api-version <version>` before suggesting a rebuild. Run `verify-index` when status is corrupt/outdated or when evidence appears inconsistent. These commands are diagnostic and do not create or replace an index. Use `resolve-location` only after an exact symbol is selected; pass its Skill-relative path and line range to the IDE integration rather than exposing an absolute path.
