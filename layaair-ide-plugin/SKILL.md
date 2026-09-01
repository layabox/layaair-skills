---
name: layaair-ide-plugin
description: "Create, package, and publish LayaAir IDE editor plugins with React-first UI and current @IEditor/@IEditorEnv APIs. Use for panels, dialogs, menus, inspectors, graph/timeline editors, custom editors, build/asset plugins, scene hooks, gizmos, and installable packages. This repository is the LayaAir IDE, so editor-plugin work here uses this skill. Do NOT use this skill to operate the current IDE appearance or theme when the ThemeManagement MCP tool is available; call that tool instead."
---

# LayaAir IDE Plugin Development Skill

You are an expert at developing LayaAir IDE editor plugins (extensions). Generate production-ready TypeScript plugin code following LayaAir 3.x conventions.

## Architecture Overview

LayaAir IDE runs on Electron with **three processes**:
- **UI process**: Editor panels, menus, dialogs. Uses `@IEditor.*` decorators. Global object: `Editor`
- **Scene process**: Engine/scene logic, gizmos, build. Uses `@IEditorEnv.*` decorators. Global object: `EditorEnv`
- **Preview process**: Runtime preview. No Node.js, uses Laya engine only.

Scripts compile to: `bundle.editor.js` (UI), `bundle.scene.js` (Scene), `bundle.js` (Preview).

## Plugin Location

Plugins are created in **user projects**, not in the IDE source. Plugin scripts can be placed **anywhere under the project's `assets/` directory** — there is no required directory structure. The IDE automatically compiles scripts with `@IEditor.*` or `@IEditorEnv.*` decorators into the corresponding bundles.

Place editor-only resources (icons, styles, locales) under a plugin-specific subdirectory such as `editorResources/my-plugin/`; files under `editorResources/` are excluded from the game build. Do not place plugin resources directly in the shared `editorResources/` root, because names can collide. The physical directory containing `editorResources/` may be anywhere under `assets/`. When using a relative editor-resource path with an editor API that supports it, omit every physical prefix segment and begin at `editorResources/`, for example `editorResources/my-plugin/icon.svg`. This logical relative path is not suitable for Node.js filesystem IO.

## Sharing and Publishing Plugins

Choose the distribution form according to how the recipient should consume the plugin:

1. **Direct folder copy** is the simplest form of source sharing. A recipient can copy the plugin folder anywhere under the target project's `assets/` directory; the IDE imports its assets and compiles the decorated plugin scripts automatically. Keep the plugin self-contained, and preserve its `.meta` files when UUID-based references must remain stable.
2. **Regular resource package** is convenient for sharing the same asset content as one `.layapkg` file. Export the plugin folder with the IDE's **Export Resource Package** command, or run `layaair export-package assets/MyPlugin -o output/MyPlugin.layapkg`. The recipient imports it as project assets. Use `--include-dependencies` only when the package must also collect referenced assets outside the selected folder. This does not make the result an installable package.
3. **Installable package** is the advanced form for Package Manager installation, dependency resolution, and versioned distribution. Export exactly one project folder with the IDE's **Export Installable Package** command, or run `layaair export-installable-package packages/MyPlugin -o output/MyPlugin.layapkg`. The selected folder may be outside `assets`, but it must be inside the project and directly contain a valid `package.json`; its contents are written to the archive root.

An installable plugin package requires non-empty `name` and `version` strings. Use a stable, globally unique package name and prefer a semantic version. The manifest may also declare:

- `pluginDependencies`: maps package names to required versions or supported package sources. The Package Manager resolves and installs these LayaAir plugin packages and orders dependencies before the consuming package. Use this field for plugin-package relationships; ordinary npm `dependencies` have a different purpose.
- `contributes`: declares package capabilities consumed by IDE subsystems. The current public contribution is `contributes.engine`, an array that adds engine libraries or add-ons to existing libraries for Project Settings and build selection. This field does not register editor panels, menus, or Scene/UI scripts; continue using the appropriate `@IEditor.*` and `@IEditorEnv.*` decorators for those.
- `precompile`: a non-empty array of source-directory paths relative to the package root. During export, the IDE precompiles TypeScript in those directories, removes the listed source directories from the staged package, and writes the generated UI and Scene bundles to `build~/bundle.editor.js` and `build~/bundle.scene.js`. Every entry must name an existing directory inside the package; absolute paths and paths that escape the package root are invalid. Precompilation runs only when exactly one folder is exported and that folder directly contains `package.json`. It does not generate the Preview/runtime `bundle.js`, so do not place gameplay runtime source in a precompiled directory when the installed package must expose that source to Preview builds.

```json
{
  "name": "com.example.my-plugin",
  "version": "1.0.0",
  "precompile": ["editor", "scene"],
  "pluginDependencies": {
    "com.example.shared-tools": "1.2.0"
  },
  "contributes": {
    "engine": [
      {
        "name": "example.engine-module",
        "caption": "Example Engine Module",
        "files": ["engine/libs/example-module.js"]
      }
    ]
  }
}
```

Do not spell the CLI command as separate words or substitute a generic package command: its exact name is `export-installable-package`.

## Step-by-Step: Creating a Plugin

### 1. Ask the user what type of plugin they need

Common plugin types:
- **Panel plugin**: React editor panel (most common)
- **Menu plugin**: Add menu items to the editor
- **Inspector plugin**: Custom property fields in Inspector
- **Graph plugin**: Port-based node graphs with `IEditor.Flow`, or state-machine graphs with `IEditor.StateGraph`
- **Timeline plugin**: Track, keyframe, event, curve, or interval editing with `IEditor.Timeline`
- **Build plugin**: Extend the build pipeline
- **Asset plugin**: Custom asset types with import/export/preview
- **ScriptableObject data asset**: Typed `.sco` data resources using the built-in 3.4.1+ workflow
- **Scene hook plugin**: React to scene events (node creation, save, etc.)
- **Gizmo plugin**: Custom scene view drawing (2D/3D)

### 2. Create the plugin following these patterns

Read `references/api-patterns.md` for the complete API reference with code examples for each plugin type.

## Key Rules

1. **Decorator placement matters**: `@IEditor.*` = UI process only, `@IEditorEnv.*` = Scene process only
2. **Editor UI defaults to React**: Build panels, dialogs, settings, and previews with `IEditor.ReactDOM` and `IEditor.React`. Do not generate widget packages or programmatic non-React UI unless the user explicitly requests legacy compatibility.
3. **Panel class** must extend `IEditor.EditorPanel`; create an `IEditor.ReactDOM`, mount it in `this._panel`, render JSX, and call `dispose()` in `onDestroy()`
4. **Dialog class** must extend `IEditor.Dialog<IEditor.ReactDOM>`, assign an `IEditor.ReactDOM` to `this.contentPane`, render JSX, and dispose it with the dialog
5. **Inspector field** must extend `IEditor.PropertyField` and implement `create()` + `refresh()`; prefer the built-in React inspector components for plugin-owned forms
6. **Build plugin** must implement `IEditorEnv.IBuildPlugin` interface
7. **Settings location**: `"project"` (shared), `"local"` (gitignored), `"application"` (global), `"memory"` (transient)
8. **editorResources namespace and path forms**: Assets here are NOT published to the game build. Always create a plugin-specific subdirectory such as `editorResources/my-plugin/`; never put plugin files directly in the shared root. If an editor API supports relative editor-resource paths, begin the value with `editorResources/` and strip every directory before that segment. Do not pass this logical relative path to Node.js filesystem APIs. For Node.js IO, resolve it with `const asset = await Editor.assetDb.getAsset("editorResources/...", true)`—the second argument is required—then obtain the absolute path with `Editor.assetDb.getFullPath(asset)`.
9. **Cross-process calls**: UI calls Scene via `Editor.scene.runScript("ClassName.method", ...args)`, Scene calls UI via `EditorEnv.sendMessageToPanel("PanelName", "method")`
10. **React runtime, components, graphs, and Timeline**: React is built in; import it without installing the runtime. Use `IEditor.React` for editor-integrated controls, including `CodeEditor`, `DiffEditor`, and `HighlightedCode`; use `IEditor.Flow` for port-based node graphs, `IEditor.StateGraph` for pinless state-machine graphs, and `IEditor.Timeline` for track/key/event/range editing. `StateGraphEditor` reports rejected connection attempts through `onConnectRejected`. These are sibling runtime namespaces; use `IEditor.IFlow`, `IEditor.IStateGraph`, and `IEditor.ITimeline` for their TypeScript interfaces. Read `references/api-patterns.md` §2 before generating UI so the current APIs and props are used.
11. **Tool buttons with tips**: If a toolbar/icon button needs tips, use `<ToolButton title="...">`; do not put a native `title` on `<button>` for that purpose. `ToolButton` consumes `title`, uses the editor tooltip system, suppresses Chromium's native tooltip, and groups adjacent tooltips for fast traversal. A tool button without tips can use a normal `<button>`. Use `TooltipTarget` for tips on non-button elements.
12. **Theme workflow**: `IEditor.ReactDOM` injects both dark and light theme tokens plus the base component styles. Prefer semantic CSS variables and built-in classes; do not hard-code dark colors. Use `--toggle-button-selected-bg` / `--toggle-button-selected-text` for persistent `aria-pressed` states and the current `--timeline-*` tokens for Timeline skins. There are no layout utility classes, so use inline layout styles or a small imported CSS file passed to `adoptStyles()`. Read the theme source/type declarations when a token is not documented.
13. **Standalone icons across themes**: If the same standalone monochrome/editor icon must work in both dark and light modes, apply `filter: var(--ui-icon-filter);` to its custom `<img>`, standalone `EditorImage`, or background-image style. The token is `none` in dark mode and darkens the icon in light mode. Built-in toolbar/icon selectors already apply it to their nested `EditorImage`; do not apply it twice.
14. **TypeScript config**: Must have `"experimentalDecorators": true` and `"jsx": "react-jsx"`. React runtime is built in, but TypeScript compilation needs `"@types/react"` and `"@types/react-dom"` in `devDependencies`.
15. **Images**: In React, `import icon from "./icon.png"` returns an absolute `file://` URL string; use it in JSX or CSS. Supported formats: png, jpg, gif, svg, webp, ico, bmp. Keep editor-only images under the plugin's own `editorResources/<plugin-name>/` directory.
16. **IFrame**: Never use a raw `<iframe>`. Create one `IEditor.WebIFrame` with `useRef` + a callback ref, append its `.element` to a React container, and hide with `display:none` instead of removing it from the DOM.
17. **Node.js modules**: Node built-in modules (`fs`, `path`, `child_process`, etc.) are available in UI/Scene code and can be loaded with `import` or `require()` without installing packages. They are unavailable in Preview code.
18. **Renaming scripts**: When renaming a script file, always rename the paired `.meta` file so its asset UUID remains attached.
19. **Type name uniqueness**: Type names passed to `Editor.typeRegistry.addTypes` and `InspectorPanel.inspect` (the `name` field) share the global type registry and must be unique. Use a plugin-specific prefix, such as `"MyPlugin_SettingsType"`.
20. **Panel ID uniqueness**: The `id` passed to `@IEditor.panel(id, ...)` is globally unique. Use a plugin- or company-specific prefix, such as `"MyCompany.ProjectManager.MainPanel"`.
21. **Type caption localization**: Write English type/property labels directly in `caption`; only Chinese needs an additional translation map. For manually declared `IEditor.FTypeDescriptor` arrays, attach `captionTranslation` before `Editor.typeRegistry.addTypes()`. For plugin script types generated from `@Laya.regClass()` / `@Laya.property()`, do not add duplicate descriptors: key translations by the script asset UUID, apply them to `Editor.typeRegistry.types`, and reapply on `onUserTypesChanged`. In both forms, `"#"` is the type caption and property-name keys are property captions. Treat translations as already loaded; their storage path is not fixed. See `references/api-patterns.md` §18.
22. **Menu instance lifecycle**: Never call anonymous `IEditor.Menu.create([...])` inside click, pointer, context-menu, or other repeated interaction handlers. Cache the menu instance, or lazily reuse a globally unique plugin-prefixed ID with `IEditor.Menu.getById(id) ?? IEditor.Menu.create(id, template)`. Repeated `create()` with the same ID throws. Give menu items stable IDs; update the reused menu with `setItems()`, `setItemEnabled()`, `setItemVisible()`, `setItemChecked()`, or `setItemLabel()`, then call `show()`. See `references/api-patterns.md` §3.
23. **Code UI**: Use `IEditor.React.CodeEditor` for editable code, `DiffEditor` for read-only unified or side-by-side differences, and `HighlightedCode` for read-only snippets; do not install or bundle CodeMirror or highlight.js. `CodeEditor` is controlled, uses `fileName` to select language support, and reports the platform save shortcut (`Mod-S`) through `onSave`. Prefer `DiffEditor` over building highlighted diff HTML, especially for large files. Use `highlightElement` only for existing imperative DOM. These APIs install their own editor-integrated styling.
24. **React style/root helpers**: Keep panel-level CSS on `ReactDOM.adoptStyles()`. Use `IEditor.React.useStyles()` for reusable component-owned CSS, `useDOMRoot()` when an imperative library must follow a ReactDOM root across windows, and `ensureStyles()` for permanent imperative-module CSS. Give every `ensureStyles()` registration a stable plugin-prefixed ID; the first registration for an ID wins in each root.
25. **ScriptableObject `.sco` assets (3.4.1+)**: For typed serializable data resources, extend `Laya.ScriptableObject`, register the class with `@Laya.regClass()`, expose fields with `@Laya.property()`, and use `@Laya.classInfo({ menu, newAssetName, icon })` to add it to Project/Create. The editor already supplies `.sco` import, Inspector editing, saving, dependency analysis, export, and loading; do not register a custom asset importer/saver/loader for this format. Preserve the script `.meta`, because the serialized `_$type` identifies the registered script type. See `references/api-patterns.md` §19.

## Full API Reference

For the complete API beyond what `references/api-patterns.md` covers, read the type declaration files in the project's `engine/types` directory:
- **editor.d.ts** — UI process API (`IEditor` namespace, global `Editor` object)
- **editor-env.d.ts** — Scene process API (`IEditorEnv` namespace, global `EditorEnv` object)
- **LayaAir.d.ts** — engine/runtime APIs, including `Laya.ScriptableObject`, `Laya.classInfo`, `Laya.property`, and resource loading
- **IReactComponents / `IEditor.React` declarations** — current public React components, code editor/highlighting APIs, props, theme/style/root helpers, and interaction utilities
- **IFlow / `IEditor.IFlow` declarations** — port-based graph data, registries, store, commands, and `IEditor.Flow` runtime values
- **IStateGraph / `IEditor.IStateGraph` declarations** — state-machine nodes, edges, callbacks, and `IEditor.StateGraph` runtime values
- **ITimeline / `IEditor.ITimeline` declarations** — mutable Timeline documents, view state, actions, split handles, snapshots, commands, range/key/event types, and `IEditor.Timeline` runtime values

## Output Format

When creating a plugin, always:
1. Place plugin files under the user project's `assets/` directory; ask only when the intended location cannot be inferred
2. Include proper decorators and type annotations
3. Put editor-only resources (icons, styles, locales) in a plugin-specific `editorResources/<plugin-name>/` directory, never directly in the shared root; use `editorResources/...` only as a logical relative path for supporting editor APIs, and resolve an absolute path before Node.js filesystem IO
4. Add i18n support if the plugin has user-visible strings
5. Use React for editor UI and explain which process each file runs in (UI vs Scene)
