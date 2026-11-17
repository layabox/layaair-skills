---
name: layaair-api-knowledge
description: Query and verify versioned official LayaAir engine and LayaAirIDE APIs plus local scene node and component serialization schemas bundled with the Skill. Use whenever answering, writing, modifying, debugging, or reviewing code that depends on LayaAir APIs, version differences, IDE extension APIs, node/component types, serialized properties, defaults, ranges, Inspector metadata, or schema inheritance. Use LayaAirIDE MCP for edits to an open scene or prefab. Query schemas when an edit depends on serialized node or component properties, including when passing those properties to a live editing API. Do not use for tasks unrelated to LayaAir APIs or scene schemas.
---

# LayaAir API Knowledge

Use the LayaAir MCP tools first to search and verify LayaAir APIs. Use the bundled query script only when the relevant MCP tool is unavailable.

For a live scene or prefab edit, start with `SceneManagement` using `action: "getCurrent"` and `params: {includeState: true}`. Use an existing editing tool when it covers the change, or `ExecutePluginScript` with the public `Editor.sceneManager.activeScene` API. If you need a live-editing example, call `LayaAir_GetIdeGuide` with `topic: "scene-editing"`, or read "Editing the Active Scene" in the available `layaair-ide-plugin` reference. Reuse signatures already in context and query serialized fields only when the edit depends on them. Use `SceneManagement` with `action: "save"` when the change should be persisted.

## Resolve the tool and version

For script fallback, locate `scripts/layaair-api.cjs` relative to this `SKILL.md`:

```text
node "<skill-dir>/scripts/layaair-api.cjs" <command> [value] [options]
```

API declarations under `data/engine/version/<version>/` are build-time inputs and are not part of the published runtime Skill. Runtime queries use the verified SQLite cache at `layaair-api-knowledge/layaair-api.sqlite`; the SQLite cache is downloaded separately and is not part of the imported Skill.

LayaAir Agent, the LayaAir MCP plugin, and this Skill share a per-user cache independent of the IDE. The default root is `~/Library/Application Support/LayaAir` on macOS, `%LOCALAPPDATA%\LayaAir` on Windows, and `${XDG_DATA_HOME:-~/.local/share}/LayaAir` on Linux. The same path is used when the IDE is closed or absent. For an isolated run, use `--cache-root <path>` or set `LAYA_API_KNOWLEDGE_CACHE_ROOT` for all participating processes. The private generator uses the project-local `library` directory by default.

Pass `api_version` to MCP or `--api-version <version>` to the script whenever the requested version is known. The MCP plugin supplies the current IDE version by default for versioned queries. Without an explicit version, the script first reads `library/project-version.txt` in the project root; only when that file is missing does it read the uniquely identified `*.laya` descriptor version. Invalid version files report an error. Without a project, it falls back to the newest available cached version.

For runtime API queries, an exact version wins; otherwise the tool selects the nearest available declaration snapshot under its version-selection policy: the newest cached version no higher than the request, or the oldest available version if all cached versions are newer. The knowledge base maintains these snapshots as compatible API sources for the requested target version. Use the automatically selected snapshot as authoritative API evidence for that target; no separate per-version compatibility allowlist is required.

Inspect `versionSelection` to distinguish the requested target (`requestedVersion`) from the declaration snapshot label (`apiVersion`). `fallback: true` means that a differently labeled snapshot was selected, not that compatibility is unknown or unverified. For example, a request for 3.4.2 that selects 3.4.0 uses the knowledge base's compatible APIs for 3.4.2. Answer for the requested target without a version-mismatch disclaimer. When the user asks about provenance, report both versions accurately. Type checks and IDE loading validate the generated code, not whether the knowledge base supports the target. Version comparisons and index diagnostics remain exact; component schemas remain unversioned.

Use `LayaAir_ListApiVersions` to list available versions. The script supports version listing and index diagnostics.

```text
node "<skill-dir>/scripts/layaair-api.cjs" versions
node "<skill-dir>/scripts/layaair-api.cjs" status --api-version 3.4.0
node "<skill-dir>/scripts/layaair-api.cjs" verify-index --api-version 3.4.0
```

The MCP plugin checks the remote manifest after startup and every six hours; a 24-hour TTL prevents unnecessary requests. To diagnose or force a check manually:

```text
node "<skill-dir>/scripts/layaair-api.cjs" cache-update --cache-root "<user-cache-root>"
node "<skill-dir>/scripts/layaair-api.cjs" cache-update --cache-root "<user-cache-root>" --force
```

The default cache manifest is `https://ldc-1251285021.file.myqcloud.com/layaair-api-knowledge/stable/manifest.json`. Manifest checks use cache-busting requests while versioned artifacts remain immutable. Remote API versions and declaration hashes are not pinned to this Skill release. Each process downloads and checks size, SHA-256, SQLite integrity, and schema independently. Publication takes a short lock, compares manifest `publishedAt`, and atomically replaces the active database. Queries read the published database during downloads. Never treat an update failure as a query failure when a usable local cache exists.

## Choose a script fallback mode

When the relevant MCP tool is unavailable, choose how to run the script for the task. Use the CLI for occasional lookups or when your tools cannot keep a Node.js IPC session open. For several related queries, you can use the [resident query process](references/query-playbook.md#reuse-a-query-process) to reuse the open SQLite database. Keep it open only while you need it. Both modes use the same cache and query options.

## Query workflow

Use the named MCP tools when available. The API lookup commands below show the script fallback.

1. Determine whether the task targets game runtime APIs or LayaAirIDE extension APIs. Before starting a game UI coding task, call `ProjectManagement` with `action: "getProjectInfo"` and read its current `modules` and `addons`. If MCP is unavailable, read `settings/PlayerSettings.json` in the target project. Refresh this context for each UI task and after the user changes project settings; do not reuse an earlier conversation snapshot.
   - Select the UI implementation before searching for a concrete API. When `addons["laya.ui"]` is `"ui2"`, use new UI types such as `Laya.GImage` with `src`, then query their signatures. Finding `Laya.Image` in the knowledge base does not make it applicable to this configuration.
   - Check whether the UI module is enabled. For mixed implementations (`"both"`), follow the target UI hierarchy and task requirements. If configuration is missing or ambiguous, inspect the target scene and project settings before choosing a UI family.
2. If the declaration name is unknown or an exact lookup returns `NOT_FOUND`, use `LayaAir_SearchApi` with a short symbol name and the relevant domain or known owner. Use the qualified name from the results for exact lookup.

   ```text
   node "<skill-dir>/scripts/layaair-api.cjs" search "Sprite" --api-version 3.4.0 --domain engine-runtime
   node "<skill-dir>/scripts/layaair-api.cjs" search "ISceneManager" --api-version 3.4.0 --domain ide
   ```

3. When the qualified name is already known, go directly to `LayaAir_GetApi`. Fetch exact signatures only when you need details missing from the guide or search results.

   ```text
   node "<skill-dir>/scripts/layaair-api.cjs" get "Laya.Sprite" --api-version 3.4.0
   ```

4. When discovering members, use `LayaAir_GetApiMembers` with `detail: "summary"` and a focused limit. Reuse the returned signatures. Request inherited members or call `LayaAir_GetApiInheritance` when inheritance matters.

   ```text
   node "<skill-dir>/scripts/layaair-api.cjs" members "Laya.Sprite" --api-version 3.4.0 --inherited --limit 100
   node "<skill-dir>/scripts/layaair-api.cjs" inheritance "Laya.Sprite" --api-version 3.4.0 --direction ancestors
   ```

5. Use the same tools to inspect LayaAir types referenced in parameters, properties, or return values.

6. Use `LayaAir_CompareApiVersions` when a task concerns an upgrade or compatibility.

   ```text
   node "<skill-dir>/scripts/layaair-api.cjs" compare-versions --from-version 3.4.0 --to-version 3.4.1 --limit 100
   node "<skill-dir>/scripts/layaair-api.cjs" compare-versions "Laya.Sprite" --from-version 3.4.0 --to-version 3.4.1 --diff-type method --domain engine-runtime
   ```

   Omit the symbol name to compare the selected API scope. Use `--diff-type class|method|property|all` to restrict the result. The response distinguishes added, removed, and modified stable API identities; a modified entry lists changes to signature, documentation, or deprecation state.

7. Base the answer and generated code only on returned signatures and relationships.
8. Answer for the requested target version using the selected compatible snapshot. Explain snapshot selection only when the user asks about versions or provenance, or when diagnosing the knowledge index.
9. If no symbol or type reference is returned, report it as unresolved in the selected version. Never invent a replacement API.

## Node and component schemas

When reading or editing `.ls` or `.lh` JSON, read [references/scene-prefab-format.md](references/scene-prefab-format.md) for hierarchy structure, prefab overrides, and resource/object references. Query concrete properties through the schema commands below; explicit file validation uses `Laya_ValidateLayaAsset` and the IDE official hierarchy validator.

- New node `_$id` values, including prefab instance roots, must be random 8-character strings from `0-9a-z`; non-singleton components that need their own IDs use random 4-character strings from the same alphabet. Prefer IDE allocation; when authoring JSON directly, run a random generator and check for collisions in the owning hierarchy. Put semantic labels in `name`, never derive IDs from names, paths, roles, or counters.
- Preserve existing IDs and the actual targets of `_$ref`, `_$override`, and `_$parent`, including nested paths. Do not regenerate existing IDs to match the new-ID convention or reuse the example IDs. Component schemas describe properties, not these hierarchy identity rules; successful validation does not establish that an ID was randomly generated.

Use the local component schema dataset when the task concerns scene/prefab node types, component properties, serialized defaults, value ranges, enum sources, Inspector metadata, or which node classes derive from a base. This is distinct from TypeScript API lookup.

```text
node "<skill-dir>/scripts/layaair-api.cjs" schema-search "Camera" --limit 20
node "<skill-dir>/scripts/layaair-api.cjs" schema-search "physics" --world-type 3d
node "<skill-dir>/scripts/layaair-api.cjs" schema-get "Camera" --inherited
node "<skill-dir>/scripts/layaair-api.cjs" schema-descendants "Node" --max-depth 2 --limit 100
```

- Run `schema-search` when the exact node/component name is unknown. It searches names and descriptions.
- Run `schema-get` before generating serialized node/component data. Add `--inherited` to receive the parent chain and flattened `effectiveProperties`, where each property identifies `declaredBy`.
- Run `schema-descendants` to discover derived node/component types. Add `--include-properties` only when full subclass schemas are needed.
- Treat `data/json/component_schemas.json` as an unversioned shared dataset. Do not pass `--api-version` to schema commands or claim that its evidence belongs to a specific engine declaration version.
- Schema lookup is read-only discovery. Use LayaAirIDE MCP tools to create or modify actual scenes, prefabs, nodes, components, and project resources.

## Code validation

LayaAir IDE preinstalls the `tsc` command; no additional TypeScript installation is needed. Use it to type-check generated or edited code with the project's TypeScript configuration:

```text
tsc --noEmit -p tsconfig.json
```

Run the command from the project root, or pass the relevant configuration path to `-p`. IDE builds use esbuild and do not provide TypeScript type diagnostics.

Resolve declaration evidence with `LayaAir_ResolveApiLocation` when needed. Use the script command if the MCP tool is unavailable.

```text
node "<skill-dir>/scripts/layaair-api.cjs" resolve-location "Laya.Sprite.pos" --api-version 3.4.0
```

Use only the returned Skill-relative path and line numbers. Never expose or construct a machine-absolute path in an answer.

## Domain rules

- Use `--domain engine-runtime` for game code and runtime engine APIs.
- Use `--domain ide` for LayaAirIDE plugins, editor processes, resource tools, and `IEditor` APIs.
- Use `--domain support` only for auxiliary CSS/GLSL declarations.
- Do not mix an IDE-only API into runtime code even when its name matches.

## IDE API execution boundary

- Use the `ide` domain to answer questions about, generate, debug, or review LayaAirIDE plugin and MCP implementation code.
- For an ordinary request to create or modify nodes, scenes, prefabs, or project resources, use LayaAirIDE MCP tools.
- Treat this Skill as API discovery and verification only. Finding an Editor API does not provide a channel to execute it against the current IDE.
- Call Editor APIs directly only in code intended to run inside the appropriate LayaAirIDE plugin or editor-process context.

## Query options and references

- Use `--api-version <version>` to isolate every query to one bundled declaration version.
- Use `--kind class`, `--kind method`, `--kind property`, or another exact symbol kind to reduce ambiguity.
- Use `--owner <FQN>` to search only direct members of one class or interface.
- Use `--limit N` for bounded results; the hard search maximum is 100.
- For `compare-versions`, use `--from-version` and `--to-version`; do not combine it with `--api-version`.
- Use `--world-type 2d|3d` only with `schema-search`.
- Prefer fully qualified names such as `Laya.Sprite` after candidate discovery.
- Read [references/query-playbook.md](references/query-playbook.md) for ambiguity, overload, inheritance, and failure handling.
- Read [references/output-contract.md](references/output-contract.md) when interpreting tool JSON or diagnosing an index problem.

## Evidence rules

Treat `apiVersion`, `sourceDomain`, `sourcePath`, `sourceSha256`, `startLine`, and `endLine` as internal authoritative evidence.

- By default, answer with the verified API facts without displaying declaration paths, line numbers, hashes, SQLite/cache details, or a separate evidence footer.
- Keep declaration snapshot labels internal by default. When the user asks which version was queried or when diagnosing the knowledge index, distinguish the requested target from the actual selected snapshot; never relabel snapshot evidence as an exact-version query.
- If provenance is useful, say only that the result was verified through the local LayaAir API knowledge base. Do not imply that the answer bypassed the knowledge base and read project declaration files directly.
- Show declaration locations only when the user explicitly asks for source evidence or when diagnosing the knowledge index. In that case, use only the returned Skill-relative path and line numbers.
