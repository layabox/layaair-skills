---
name: layaair-ide-plugin
description: "Create LayaAir IDE editor plugins (extensions). TRIGGER when: user asks to create/build/write a plugin, panel, editor panel, menu, inspector field, build plugin, asset processor, custom editor, scene hook, gizmo, or any editor extension. Also trigger when user mentions LayaAir/Laya/laya, @IEditor, @IEditorEnv, EditorPanel, panel/面板, or wants to extend the IDE. Chinese triggers: 编辑器插件, 编辑器面板, 插件面板, 面板插件, 写一个面板, 写个插件. This project IS the LayaAir IDE — any request to write an editor plugin or panel in this repo should use this skill."
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

One convention: place editor-only resources (icons, widgets, locales) in an `editorResources/` folder — files there are excluded from the game build.

## Step-by-Step: Creating a Plugin

### 1. Ask the user what type of plugin they need

Common plugin types:
- **Panel plugin**: Custom editor panel (most common)
- **Menu plugin**: Add menu items to the editor
- **Inspector plugin**: Custom property fields in Inspector
- **Build plugin**: Extend the build pipeline
- **Asset plugin**: Custom asset types with import/export/preview
- **Scene hook plugin**: React to scene events (node creation, save, etc.)
- **Gizmo plugin**: Custom scene view drawing (2D/3D)

### 2. Create the plugin following these patterns

Read `references/api-patterns.md` for the complete API reference with code examples for each plugin type.

## Key Rules

1. **Decorator placement matters**: `@IEditor.*` = UI process only, `@IEditorEnv.*` = Scene process only
2. **Panel class** must extend `IEditor.EditorPanel` and implement `async create()` returning `this._panel`
3. **Dialog class** must extend `IEditor.Dialog` and set `this.contentPane`
4. **Inspector field** must extend `IEditor.PropertyField` and implement `create()` + `refresh()`
5. **Build plugin** must implement `IEditorEnv.IBuildPlugin` interface
6. **Settings location**: `"project"` (shared), `"local"` (gitignored), `"application"` (global), `"memory"` (transient)
7. **editorResources/** directory: assets here are NOT published to the game build
8. **Cross-process calls**: UI calls Scene via `Editor.scene.runScript("ClassName.method", ...args)`, Scene calls UI via `EditorEnv.sendMessageToPanel("PanelName", "method")`
9. **React panels**: React is built-in to the IDE — just `import` and use, no `npm install` needed. Use `IEditor.ReactDOM`, call `adoptStyles()` for CSS, `makeFullSize()` to fill parent, `render(<App/>)` to mount. Only prerequisite: ensure `"jsx": "react-jsx"` is set in tsconfig (see rule 11)
10. **CSS workflow**: ReactDOM auto-injects a base dark-theme stylesheet with CSS variables, styled HTML elements (`<button>`, `<input>`, `<select>`, etc.), component classes (`.btn-primary`, `.list-item`, `.tab`, `.panel`, etc.), and layout utilities (`.flex`, `.gap-2`, `.p-2`, etc.). **Always prefer the built-in theme first** — most panels need zero custom CSS. Only create a custom CSS file when the built-in classes are insufficient. `import styles from './Foo.css'` returns CSS as a string (via built-in esbuild css-text plugin), pass to `reactDOM.adoptStyles(styles)`. See `api-patterns.md` §2 "Built-in Theme Reference" for the full variable/class list.
11. **TypeScript config**: Must have `"experimentalDecorators": true` and `"jsx": "react-jsx"` (for React)
12. **Images**: Two approaches depending on UI mode. **React panels**: `import icon from './icon.png'` returns an absolute `file://` URL string, use in JSX `<img src={icon} />`; CSS `url(./icon.png)` also auto-resolved (supported: png, jpg, gif, svg, webp, ico, bmp). **Built-in UI (FairyGUI)**: use `editorResources/` paths directly, e.g. `"editorResources/my-plugin/icon.svg"`. Editor-only images should always go in `editorResources/` to exclude from game build.
13. **IFrame**: Never use raw `<iframe>` HTML element. Always use `IEditor.WebIFrame` instead. In React, create the instance once via `useRef` + `useCallback`, append its `.element` to a container div. When hiding, use `display:none` — do NOT remove from DOM (avoids reload/state loss).
14. **Node.js modules**: Node built-in modules (`fs`, `path`, `child_process`, etc.) are available in the IDE — just `import` and use, no install needed.
15. **Renaming scripts**: When renaming a script file, always rename the corresponding `.meta` file as well (e.g. rename `Foo.ts` → `Bar.ts`, must also rename `Foo.ts.meta` → `Bar.ts.meta`). The `.meta` file stores the asset UUID and must stay paired with its file.
16. **Type name uniqueness**: `Editor.typeRegistry.addTypes` 和 `InspectorPanel.inspect` 中传入的类型名（`name` 字段）在全局类型注册表中必须唯一，否则会与其他插件冲突。命名时加上插件专属前缀，例如 `"MyPlugin_SettingsType"` 而非 `"SettingsType"`。
17. **Panel ID uniqueness**: `@IEditor.panel(id, ...)` 的 `id` 是全局唯一标识符，简短通用的名字（如 `"MyPanel"`、`"Settings"`）容易与其他插件冲突。建议使用带插件/公司前缀的复杂名字，例如 `"MyCompany.ProjectManager.MainPanel"` 而非 `"MainPanel"`。

## Full API Reference

For the complete API beyond what `references/api-patterns.md` covers, read the type declaration files in the project's `engine/types` directory:
- **editor.d.ts** — UI process API (`IEditor` namespace, global `Editor` object)
- **editor-env.d.ts** — Scene process API (`IEditorEnv` namespace, global `EditorEnv` object)
- **editor-ui.d.ts** — Editor UI library (`IEditorUI` namespace, FairyGUI components)

## Output Format

When creating a plugin, always:
1. Ask the user where to place the plugin files (anywhere under project `assets/` is valid)
2. Include proper decorators and type annotations
3. Put editor-only resources (icons, widgets) in `editorResources/` directory
4. Add i18n support if the plugin has user-visible strings
5. Explain which process each file runs in (UI vs Scene)
