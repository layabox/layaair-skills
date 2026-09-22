# LayaAir IDE Plugin API Patterns Reference

Complete code examples for all plugin types. Read the relevant section based on what the user needs.

> **Important: Type Name Uniqueness**
> 
> `Editor.typeRegistry.addTypes` 和 `InspectorPanel.inspect` 中的类型名（`name` 字段）注册到全局类型表，必须唯一。命名时加上插件专属前缀，如 `"MyPlugin_ConfigType"` 而非 `"ConfigType"`。
> 
> 下面示例中的类型名仅作演示，实际开发时请替换为带前缀的名称。

> **Important: Namespace editor resources**
>
> Create a plugin-specific subdirectory such as `editorResources/my-plugin/` and keep all plugin icons, styles, locales, and other editor-only assets inside it. Do not place plugin resources directly in the shared `editorResources/` root, where names can collide with other plugins.
>
> The physical prefix before `editorResources/` may be anywhere under `assets/`. When an editor API supports a relative editor-resource path, strip that prefix and begin the relative value at `editorResources/`:
>
> ```text
> Physical: assets/plugins/my-plugin/editorResources/my-plugin/icon.svg
> Relative: editorResources/my-plugin/icon.svg
> ```
>
> That relative value is an editor resource locator, not a Node.js filesystem path. For Node.js IO, resolve it to an absolute path in the UI process:
>
> ```ts
> const asset = await Editor.assetDb.getAsset(
>     "editorResources/my-plugin/icon.svg",
>     true
> );
> const absolutePath = Editor.assetDb.getFullPath(asset);
> ```
>
> The `true` argument is required when resolving an `editorResources/...` locator. `getAsset()` returns `IAssetInfo`; `getFullPath()` converts that asset to the absolute path required by Node.js filesystem APIs.

## Table of Contents
1. [Panel Plugin (React)](#1-panel-plugin)
2. [React UI Guide](#2-react-ui-guide) — includes built-in controls, code editing/diffs/highlighting, style/root helpers, `IEditor.Flow`, `IEditor.StateGraph`, `IEditor.Timeline`, and the Built-in Theme Reference
3. [Menu Plugin](#3-menu-plugin)
4. [Dialog](#4-dialog)
5. [Inspector Field](#5-custom-inspector-field)
6. [Inspector Layout (Asset Config)](#6-inspector-layout)
7. [Settings & Preferences](#7-settings--preferences) — includes `IEditor.SecretStorage` for API keys and authenticated HTTP
8. [Build Plugin](#8-build-plugin)
9. [Custom Build Target](#9-custom-build-target)
10. [Asset Type Plugin](#10-custom-asset-type)
11. [Asset Import/Export](#11-asset-importexport)
12. [Asset Thumbnail & Preview](#12-asset-thumbnail--preview)
13. [Asset Processor](#13-asset-processor)
14. [Scene Hook](#14-scene-hook)
15. [Custom Editor (Gizmos)](#15-custom-editor-gizmos)
16. [Asset Database API](#16-asset-database-api)
17. [Cross-Process Communication](#17-cross-process-communication)
18. [I18n Support](#18-i18n-support)
19. [ScriptableObject `.sco` Data Assets](#19-scriptableobject-sco-data-assets)
20. [Package Precompilation](#20-package-precompilation)
21. [Frame Debugger & Profiler](#21-frame-debugger--profiler)

---

## 1. Panel Plugin

**Process**: UI  
**Use**: Custom React editor panel

```tsx
import { useState } from "react";
import styles from "./MainPanel.css";

const PANEL_ID = "MyCompany.MyPlugin.MainPanel";

function MainPanelView() {
    const [name, setName] = useState("");
    return (
        <main style={{ height: "100%", padding: 8 }}>
            <IEditor.React.TextInput
                value={name}
                placeholder="Name"
                onCommit={value => { setName(value); return true; }}
            />
        </main>
    );
}

@IEditor.panel(PANEL_ID, {
    title: "My Panel",
    icon: "editorResources/my-plugin/icon.svg",
    location: "right",        // "left"|"right"|"top"|"bottom"|"popup"|"embed"
    locationBase: "ScenePanel", // Reference panel for positioning
    autoStart: false,          // Auto-open on project load
    showInMenu: true,          // Show in Panel menu
})
export class MainPanel extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;

    async create() {
        this._react = new IEditor.ReactDOM();
        this._react.setSize(600, 500);
        this._react.adoptStyles(styles);
        this._panel = this._react;
        this._react.render(<MainPanelView />);
    }

    onStart() {
        // Panel becomes visible/active
    }

    onUpdate() {
        // Called every frame while active
    }

    onSelectionChanged() {
        // Editor selection changed
        let selection = Editor.scene?.getSelection();
    }

    onSceneActivate(scene: IEditor.IMyScene) {
        // Scene becomes active
    }

    onSceneDeactivate(scene: IEditor.IMyScene) {
        // Scene becomes inactive
    }

    onHotkey(combo: string): boolean {
        if (combo === "mod+s") { /* handle */ return true; }
        return false;
    }

    onDestroy() {
        this._react?.dispose();
    }
}
```

### Persisting panel view state: `onSaveStatus`

`onSaveStatus(): void` saves a panel's view state to `Editor.workspaceConf` for restoration when the panel is initialized again. Typical state includes the active tab, filter text, expanded items, zoom, and splitter widths; it is not the callback for saving edited assets or business files.

Use `Editor.workspaceConf.data.getSection(this.panelId)` to namespace the state by the globally unique panel ID. Write small serializable values in `onSaveStatus()` and read them with defaults in `onStart()`, then apply them to the React view/model. Save the latest committed UI values, not just the initial props; React-owned state needs to be available to the panel through a model, ref, or callback.

```tsx
@IEditor.panel("MyCompany.MyPlugin.FilterPanel", { title: "Filter Panel" })
export class FilterPanel extends IEditor.EditorPanel {
    private _react!: IEditor.ReactDOM;
    private _filter = "";

    async create() {
        this._react = new IEditor.ReactDOM();
        this._panel = this._react;
        this.renderView();
    }

    onStart(): void {
        const conf = Editor.workspaceConf.data.getSection(this.panelId);
        this._filter = conf.get("filter", "");
        this.renderView();
    }

    onSaveStatus(): void {
        const conf = Editor.workspaceConf.data.getSection(this.panelId);
        conf.set("filter", this._filter);
    }

    private renderView(): void {
        this._react.render(
            <IEditor.React.TextInput
                value={this._filter}
                placeholder="Filter"
                onCommit={value => {
                    this._filter = value;
                    this.renderView();
                    return true;
                }}
            />
        );
    }

    onDestroy(): void {
        this._react?.dispose();
    }
}
```

The current IDE calls `onSaveStatus` during editor shutdown and before destroying plugin panels for hot reload. It is synchronous: returned promises are not awaited, so do not perform asynchronous IO here. This hook is not a per-edit autosave or a guarantee for every hide/close action; if a particular interaction must persist immediately, update the workspace section at that interaction too.

Keep responsibilities separate: `onSave(): Promise<void>` saves actual edited files, `getUnsavedFiles()` reports them for the close/save confirmation, and `onDestroy()` releases React roots and subscriptions. `onSaveStatus()` only captures view state.

### Panel with React InspectorPanel (config-driven UI)

`InspectorPanelModel` drives the existing metadata-based inspector fields while `InspectorPanel` renders them inside React.

```tsx
const CONFIG_TYPE = "MyCompany.MyPlugin.ConfigType";

@IEditor.panel("MyCompany.MyPlugin.ConfigPanel", { title: "Config Panel" })
export class ConfigPanel extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;
    private _model: InstanceType<typeof IEditor.React.InspectorPanelModel>;
    private _data: any;

    async create() {
        Editor.typeRegistry.addTypes([{
            name: CONFIG_TYPE,
            properties: [
                { name: "text", type: "string" },
                { name: "count", type: "number", min: 0, max: 100 },
                { name: "enabled", type: "boolean", default: true },
                { name: "color", type: "color" },
                { name: "asset", type: "string", isAsset: true, assetTypeFilter: "Image" }
            ]
        }]);

        this._data = IEditor.DataWatcher.watch({});
        this._model = new IEditor.React.InspectorPanelModel();
        this._model.allowUndo = true;
        this._model.inspect(this._data, CONFIG_TYPE);

        this._react = new IEditor.ReactDOM();
        this._react.setSize(500, 400);
        this._panel = this._react;
        this._react.render(<IEditor.React.InspectorPanel model={this._model} />);
    }

    onDestroy() {
        this._model?.resetInspectors();
        this._react?.dispose();
    }
}
```

---

## 2. React UI Guide

**Process**: UI  
**Use**: Default UI stack for panels, dialogs, settings, previews, and plugin-owned forms

> **React is built-in to the IDE.** Just `import { useState } from "react"` and use JSX directly — no `npm install react` needed. The only prerequisite is ensuring `"jsx": "react-jsx"` is set in the project's `tsconfig.json`.

### CSS Workflow

The IDE build pipeline has a **built-in css-text esbuild plugin** that imports `.css` files as strings. No extra setup needed.

**How it works:**
1. `import styles from './MyPlugin.css'` → returns the CSS content as a `string`
2. Pass to `reactDOM.adoptStyles(styles)` → injected into Shadow DOM via `adoptedStyleSheets`

**Best practices (priority order — prefer built-in first):**

> **Rule: Always try the built-in theme before writing custom CSS.**
> The base stylesheet already styles all standard HTML elements and provides component classes that match the editor's look. Most panels need **zero** custom CSS. Only create a custom CSS file when the built-in classes are genuinely insufficient.

| Priority | Approach | When to use | Example |
|---|---|---|---|
| 1st | **Built-in theme only** | Buttons, inputs, tabs — covers most plugins | Use `<button className="primary">`, `.tab`, `.toolbar-icon-button`, etc. Layout via inline `style` props. No CSS file needed, no `adoptStyles()` call |
| 2nd | **Custom CSS file** | Need a handful of plugin-specific styles beyond built-in | `import styles from './MyPlugin.css'` + `adoptStyles(styles)`. Use `var(--bg-base)` etc. to stay on-theme |

**Example: Custom CSS file**
```css
/* MyPlugin.css */
.container { padding: 8px; }
.card {
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 12px;
}
```

### Reusable component and imperative-module styles

Use the style API that matches ownership:

- Keep panel- or dialog-level custom CSS on `reactDOM.adoptStyles(styles)`.
- Call `IEditor.React.useStyles(styles)` at the top level of a reusable React component. The nearest `IEditor.ReactDOM` shares identical CSS in its current root and releases it after the last consumer unmounts. Keep style arrays identity-stable.
- Call `IEditor.React.useDOMRoot()` only when an imperative library needs the current `Document` or `ShadowRoot`. The hook updates if the ReactDOM moves to another editor window.
- Call `IEditor.React.ensureStyles(target, styleId, styles)` for imperative-module CSS that must remain in the target element's current root. Use a stable plugin-prefixed ID such as `"com.example.my-plugin.code-view"`; the first registration for that ID wins for the lifetime of each root.

```tsx
import controlStyles from "./MyControl.css";

function MyControl() {
    IEditor.React.useStyles(controlStyles);
    const root = IEditor.React.useDOMRoot();

    useLayoutEffect(() => {
        if (!root)
            return;
        return mountImperativeControl(root);
    }, [root]);

    return <div className="my-plugin-control" />;
}
```

Do not use `ensureStyles()` for ordinary React component CSS: it is intentionally permanent and has no unmount cleanup.

> **Important:** For most plugins, the built-in auto-styled elements (`<button>`, `<input>`, etc.) and component classes (`button.primary`, `.tab`, `.toolbar-icon-button`, etc.) are **sufficient and preferred**. They ensure visual consistency with the editor. Only create custom CSS when the built-in classes genuinely cannot cover your needs.

> **Tip:** When writing custom CSS, always reference the built-in CSS variables (`var(--bg-base)`, `var(--text)`, `var(--border)`, etc.) to keep your UI consistent with the editor theme.

### Basic React Panel

```tsx
import styles from "./MyPlugin.css";

function MyApp() {
    let [data, setData] = useState("");
    return (
        <div className="container">
            <input value={data} onChange={e => setData(e.target.value)} />
        </div>
    );
}

@IEditor.panel("MyCompany.MyPlugin.ReactPanel", {
    title: "React Panel",
    location: "right"
})
export class MyReactPanel extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;

    async create() {
        this._react = new IEditor.ReactDOM();
        this._react.setSize(600, 500);
        this._react.adoptStyles(styles);
        this._panel = this._react;
        this._react.render(<MyApp />);
    }

    onDestroy() {
        this._react?.dispose();
    }
}
```

### React with Editor Events (External Store)

```tsx
const selectionStore = IEditor.ReactDOM.createStore<any[]>([]);

function SelectionView() {
    let items = useSyncExternalStore(selectionStore.subscribe, selectionStore.get);
    return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

@IEditor.panel("MyCompany.MyPlugin.SelectionPanel", { title: "Selection" })
export class SelectPanel extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;

    async create() {
        this._react = new IEditor.ReactDOM();
        this._react.setSize(600, 500);
        this._panel = this._react;
        this._react.render(<SelectionView />);
    }

    onSelectionChanged() {
        let sel = Editor.scene?.getSelection() || [];
        selectionStore.set(sel.map(o => ({ id: o.id, name: o.name })));
    }

    onDestroy() { this._react?.dispose(); }
}
```

### Using Images

Editor-only images should live under a plugin-specific directory such as `editorResources/my-plugin/` so they are excluded from the game build and cannot collide with other plugins. Do not place them directly in the shared `editorResources/` root.

When a component or editor API accepts a relative editor-resource path, pass it from the `editorResources/` segment onward. Omit `assets/`, plugin folders, package folders, and every other physical prefix. Do not use that relative value with Node.js filesystem APIs.

Supported formats: `.png`, `.jpg`, `.gif`, `.svg`, `.webp`, `.ico`, `.bmp`.

**React panels** — `import` returns an absolute `file://` URL string:
```tsx
import icon from './icon.png';
import logo from './logo.svg';

function Header() {
    return (
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
            <EditorImage src={icon} />
            <EditorImage src={logo} />
        </div>
    );
}
```

CSS `url()` relative paths are also auto-resolved:
```css
.icon-btn {
    background-image: url(./icon.png);
    background-size: 16px 16px;
    /* Apply when this same icon must work in both dark and light modes. */
    filter: var(--ui-icon-filter);
}
```

If the same standalone icon must work in both dark and light modes, add `filter: var(--ui-icon-filter);` to its standalone `EditorImage`, custom monochrome `<img>`, or `background-image`. The token is `none` in dark mode and darkens the icon in light mode. Built-in toolbar/icon selectors already apply the filter to their nested `EditorImage`; do not apply it twice.

### Using IFrame in React

> **Never use raw `<iframe>`.** Always use `IEditor.WebIFrame`. When hiding, use `display:none` — do NOT remove from DOM (avoids reload/state loss).

```tsx
function WebView({ url, visible }: { url: string; visible: boolean }) {
    const iframeRef = useRef<IEditor.IWebIFrame | null>(null);
    const iframeContainerRef = useCallback((node: HTMLDivElement | null) => {
        if (!node) return;
        if (!iframeRef.current) {
            const iframe = new IEditor.WebIFrame();
            iframe.element.style.position = "relative";
            node.appendChild(iframe.element);
            iframeRef.current = iframe;
        } else if (!node.contains(iframeRef.current.element)) {
            node.appendChild(iframeRef.current.element);
        }
    }, []);

    useEffect(() => {
        if (iframeRef.current)
            iframeRef.current.src = url;
    }, [url]);

    // Hide with display:none, never remove from DOM
    return <div ref={iframeContainerRef} style={{ display: visible ? "block" : "none", width: "100%", height: "100%" }} />;
}
```

### Built-in React Components (`IEditor.React`)

The IDE exposes ready-made React components via `IEditor.React`. They integrate with the editor's asset database, scene, i18n system, and theme automatically — no custom CSS needed.

```tsx
// Destructure for convenience
const { EditorImage, TextInput, NumericInput, SelectInput,
        NumericInputWithSlider, RangeInput, SearchInput,
        ResourceInput, FontInput, FileInput, NodeRefInput,
        ColorInput, GradientInput, CurveInput, PolygonInput,
        CodeEditor, DiffEditor, HighlightedCode, highlightElement,
        Popup, TooltipTarget, ToolButton, LocalizedText, ResizeHandle,
        FileTabBar, FileTabBarView, InspectorPanelModel, InspectorPanel,
        beginPointerDragSession, useStyles, useDOMRoot, ensureStyles,
        getTheme, subscribeThemeChange, useThemeVersion, readCssPx,
        readCssString } = IEditor.React;
```

#### Component Overview

| Component | Description |
|---|---|
| `EditorImage` | Editor icon/image resolved from editor URL, asset UUID, file, or thumbnail URL |
| `LocalizedText` | Editor-localized text with optional UBB/HTML rendering |
| `FileTabBar` / `FileTabBarView` | Shared tab model and React view with heavy, light, and weak variants |
| `TextInput` | Text field with multiline, password, i18n translation-key, and submitOnTyping modes |
| `NumericInput` | Drag-to-edit number input — supports prefix/suffix, min/max, fractionDigits, mouse wheel |
| `NumericInputWithSlider` | `NumericInput` paired with a range slider |
| `RangeInput` | Two numeric inputs plus a dual-handle slider for an inclusive bounded range |
| `SelectInput` | Button-style dropdown with async loading and optional search |
| `SearchInput` | Search bar with leading icon and clear button |
| `ResourceInput` | Asset reference picker — drag-and-drop, copy/paste, context menu |
| `FontInput` | Project font asset picker plus system/custom font-name input |
| `FileInput` | Filesystem path input with drag/drop and native open/save dialog |
| `CodeEditor` | Controlled, lazily loaded CodeMirror editor with editor theming and filename-based language support |
| `DiffEditor` | Read-only, viewport-rendered CodeMirror diff in unified or side-by-side mode |
| `HighlightedCode` | Read-only `<code>` element highlighted with the editor's highlight.js theme |
| `NodeRefInput` | Scene node reference picker |
| `ColorInput` | Color picker popup (supports nullable/checkable) |
| `GradientInput` | Gradient editor popup |
| `CurveInput` | Curve editor popup |
| `PolygonInput` | Polygon preview/editor with optional image background and vertex limits |
| `TooltipTarget` | Wraps any element to add the editor's tooltip behavior |
| `ToolButton` | Native button semantics plus editor tooltips; use when a tool button needs tips |
| `Popup` | General-purpose anchored popup; portals into shadow root, closes on outside click/Escape |
| `ResizeHandle` | Draggable divider for resizable layouts; supports vertical/horizontal, keyboard, and double-click reset |
| `InspectorPanelModel` / `InspectorPanel` | Metadata-driven inspector model and React renderer |
| `beginPointerDragSession` | Tracks pointer drags across editor surfaces and same-origin frames |

Theme and style helpers in the same namespace: `getTheme`, `subscribeThemeChange`, `useThemeVersion`, `injectStyles`, `setTheme`, `useStyles`, `useDOMRoot`, `ensureStyles`, `readCssPx`, and `readCssString`. `useThemeVersion()` is the React hook for canvas or other imperative rendering that must recompute colors after either a theme switch or appearance-token change. Prefer reactive CSS variables for normal styling.

`IEditor.Flow`, `IEditor.StateGraph`, and `IEditor.Timeline` are also React-based, but they are separate runtime namespaces rather than members of `IEditor.React`. Use the guides below instead of nesting them below `IEditor.React`.

#### EditorImage

Resolves `editorResources/` paths, asset UUIDs, thumbnail URLs, and imported file URLs automatically.

```tsx
// Editor resource icon (16×16 by default)
<EditorImage src="editorResources/my-plugin/icon.svg" />

// Imported file URL
import icon from './icon.png';
<EditorImage src={icon} className="small-icon" />

// Asset thumbnail
<EditorImage src={Editor.assetDb.getAssetIcon(asset)} />

// Show empty slot when src is missing
<EditorImage src={maybeNull} placeholder />
```

Props: `src`, `className` (default `"small-icon"`), `iconName`, `placeholder`

#### TextInput

```tsx
<TextInput
    value={text}
    placeholder="Enter value"
    onCommit={next => { setText(next); return true; }}  // return false to reject
/>

// Multiline (auto-grows)
<TextInput value={text} multiline onCommit={next => { setText(next); return true; }} />

// With i18n translation-key editing
<TextInput value={text} multiLanguage onCommit={next => { setText(next); return true; }} />
```

Props: `value`, `onCommit(value) → boolean`, `readonly`, `multiline`, `password`, `submitOnTyping`, `multiLanguage`, `placeholder`

#### NumericInput

Supports typing, drag-to-scrub, and mouse-wheel stepping (while focused).

```tsx
<NumericInput
    value={speed}
    min={0} max={100}
    fractionDigits={1}
    suffix="°"
    onCommit={v => { setSpeed(v); }}  // return false to reject
/>

// With labeled prefix
<NumericInput value={x} prefix="X" fractionDigits={3} onCommit={v => setX(v)} />
```

Props: `value`, `onCommit(value)`, `min`, `max`, `step`, `fractionDigits`, `prefix`, `suffix`, `disabled`, `className`

#### NumericInputWithSlider

```tsx
<NumericInputWithSlider
    value={opacity}
    min={0} max={1}
    fractionDigits={2}
    onCommit={v => setOpacity(v)}
/>
```

Extra props over `NumericInput`: `sliderMin`, `sliderMax`, `centeredAtOne` (symmetric mapping around 1, for scale fields)

#### RangeInput

Use a controlled two-value tuple for a bounded inclusive range. Both number fields and both slider handles obey `min`, `max`, and `step`.

```tsx
const [range, setRange] = useState<[number, number]>([0.2, 0.8]);

<RangeInput
    value={range}
    min={0}
    max={1}
    step={0.01}
    fractionDigits={2}
    onCommit={setRange}
/>
```

Props: `value`, `min`, `max`, `onCommit(value)`, `step` (default `0.01`), `fractionDigits`, `disabled`, `className`

#### SelectInput

```tsx
const items = [
    { value: "low",    label: "Low" },
    { value: "medium", label: "Medium" },
    { value: "high",   label: "High" },
];

<SelectInput
    value={quality}
    items={items}
    onChange={(value, item) => setQuality(value)}
/>

// Async item loading on open
<SelectInput
    value={selected}
    onBeforeOpen={async () => fetchItems()}   // return SelectInputOption[] to replace items
    onChange={v => setSelected(v)}
/>
```

Props: `value`, `items`, `onChange(value, item)`, `onBeforeOpen`, `placeholder`, `disabled`, `visibleItemCount`, `searchable`, `searchPlaceholder`, `className`, `popupClassName`, `style`

#### SearchInput

```tsx
const [query, setQuery] = React.useState("");
<SearchInput value={query} onChange={setQuery} placeholder="Search..." autoFocus />
```

Props: `value`, `onChange(value)`, `placeholder`, `autoFocus`, `className`, `onKeyDown`

#### ResourceInput

Asset reference picker backed by the editor's asset database. Supports drag-and-drop from the Project panel, keyboard delete, and copy/paste of asset references.

```tsx
<ResourceInput
    value={assetId}                         // asset UUID, "res://UUID", or IAssetInfo
    typeFilter={[AssetType.Image]}          // restrict allowed types
    onCommit={(text, asset) => setAssetId(text)}
/>
```

Props: `value`, `onCommit(text, asset)`, `typeFilter`, `disabled`, `placeholder`, `className`, `allowInternalAssets`, `allowInternalGUIAssets`, `customFilter`, `onCreate`

#### FontInput

Accepts project font assets and system/custom font names. Preserve both callback values: `text` is the serializable value, while `asset` is non-null when the user selected a font asset.

```tsx
<FontInput
    value={font}
    onCommit={(text, asset) => setFont(text)}
/>
```

Props: `value`, `onCommit(text, asset)`, `disabled`, `placeholder`, `className`, `allowInternalAssets`, `allowInternalGUIAssets`

#### FileInput

Use this for a real filesystem path, not an asset reference. It supports typing, dropping a file, and opening a native open/save dialog. The default committed form is project-relative; set `absolutePath` when the value will be passed directly to Node.js filesystem APIs.

```tsx
<FileInput
    value={outputPath}
    action="save"
    absolutePath
    dialogOptions={{
        title: "Export data",
        filters: [{ name: "JSON", extensions: ["json"] }]
    }}
    onCommit={next => { setOutputPath(next); return true; }}
/>
```

Props: `value`, `onCommit(value)`, `disabled`, `absolutePath`, `action` (`"open"` or `"save"`), `dialogOptions`, `placeholder`, `className`

#### CodeEditor

Use the built-in controlled CodeMirror component for editable source or data. It loads its implementation lazily, already follows editor theme tokens, and fills the height of its parent; give the containing element a real height. Do not install CodeMirror in the plugin.

```tsx
const [draft, setDraft] = useState(source);

<div style={{ height: "100%", minHeight: 0 }}>
    <IEditor.React.CodeEditor
        content={draft}
        fileName="MyPlugin.ts"
        readOnly={saving}
        lineWrapping
        tabSize={4}
        search
        onChange={setDraft}
        onSave={() => saveSource(draft)}
    />
</div>
```

Required props are `content`, `fileName`, `readOnly`, `onChange`, and `onSave`. Keep `content` in host state and update it from `onChange`. `fileName` selects syntax support by extension; supported groups include JavaScript/JSX, TypeScript/TSX, JSON and LayaAir data files, HTML, CSS, Markdown, XML/SVG, YAML, and GLSL/HLSL/WGSL shaders. The platform save shortcut (`Mod-S`) calls `onSave`; when `readOnly` is true, editing and that callback are disabled.

Optional behavior props:

- `lineWrapping` wraps long lines; default `false`.
- `tabSize` controls indentation and tab display width.
- `indentWithTab` lets Tab indent instead of moving focus; default `true`.
- `lineNumbers` and `foldGutter` control the two gutters; both default `true`.
- `search` enables the platform search shortcut; default `false`.
- `autocompletion` enables completion UI and its keymap; default `false`.

#### DiffEditor

Use the read-only `DiffEditor` for source or data comparisons. It renders only the visible viewport and protects large inputs by reducing expensive syntax and inline-diff work, so prefer it over constructing a complete highlighted diff HTML tree.

```tsx
<div style={{ height: "100%", minHeight: 0 }}>
    <IEditor.React.DiffEditor
        before={previousSource}
        after={currentSource}
        beforeFileName="Player.ts"
        afterFileName="Player.ts"
        viewMode="side-by-side"
        collapseUnchanged={{ margin: 3, minSize: 8 }}
        ariaLabel="Player changes"
    />
</div>
```

Required props are `before`, `after`, `afterFileName`, and `viewMode` (`"unified"` or `"side-by-side"`). `beforeFileName` defaults to `afterFileName`. Use `lineWrapping` to wrap long lines. `collapseUnchanged` defaults to `{ margin: 3, minSize: 4 }`; pass `false` to show every unchanged line. Optional `className` styles the outer container, and `ariaLabel` gives the diff an accessible group label. Give the parent a real height.

#### HighlightedCode and highlightElement

Use `HighlightedCode` for a read-only snippet whose content may change. It renders a `<code>` element, writes the content as text, reapplies highlighting after changes, and installs the matching editor theme in the current root. Wrap it in `<pre>` when preformatted block layout is wanted.

```tsx
<pre className="my-plugin-code-preview">
    <IEditor.React.HighlightedCode
        content={source}
        fileName="MyPlugin.ts"
        lineWrapping
    />
</pre>
```

Props: `content`, optional highlight.js `language` or alias, optional `fileName`, optional `className`, and optional `lineWrapping` (default `false`). `language` takes precedence over `fileName`; when only `fileName` is supplied, its extension selects the language. Omitting both leaves the text unhighlighted.

For an existing imperative DOM element, put the language class and raw text on a fresh `<code>` element, then call `highlightElement`. It installs highlight styles into that element's `Document` or `ShadowRoot`; no `adoptStyles()` call is needed.

```ts
const code = document.createElement("code");
code.className = "language-typescript";
code.textContent = source;
container.appendChild(code);
IEditor.React.highlightElement(code);
```

Prefer `HighlightedCode` when React owns the element or the content changes repeatedly.

#### NodeRefInput

```tsx
<NodeRefInput
    value={nodeRef}              // IMyNode or serialized ref object
    typeFilter={["Sprite"]}      // allowed node or component type names
    onCommit={node => setNode(node)}
/>
```

Props: `value`, `onCommit(node, compType?)`, `typeFilter`, `disabled`, `className`, `onNodeResolved`

#### PolygonInput

```tsx
<PolygonInput
    value={points}                 // [x0, y0, x1, y1, ...]
    background={imageData}
    sourceWidth={512}
    sourceHeight={512}
    minPoints={3}
    previewHeight={96}
    onCommit={setPoints}
/>
```

Props: `value`, `defaultValue`, `onCommit(value)`, `background`, `sourceWidth`, `sourceHeight`, `minPoints`, `maxPoints`, `previewHeight`, `readonly`, `checkable`

#### InspectorPanelModel and InspectorPanel

Create the model once, register a globally unique type name, call `inspect`, and render the model with `InspectorPanel`.

```tsx
const model = useMemo(() => {
    const next = new IEditor.React.InspectorPanelModel();
    next.allowUndo = true;
    next.inspect(watchedData, "MyCompany.MyPlugin.SettingsType");
    return next;
}, [watchedData]);

useEffect(() => () => model.resetInspectors(), [model]);

return <IEditor.React.InspectorPanel model={model} className="inspector-scroll" />;
```

Model APIs: `inspect`, `resetInspectors`, `resetDefault`, `getInspectors`, `showCatalog`, `getScrollY`, `setScrollY`, `resizeToFit`, `allowUndo`, `history`, `onDataChanged`

#### TooltipTarget

Wraps one child element and shows an editor tooltip on hover.

```tsx
<TooltipTarget tips="Click to apply settings">
    <button className="primary" onClick={apply}>Apply</button>
</TooltipTarget>
```

Props: `tips` (string or i18n key; empty/null disables), `delay` (default 800 ms), `instantGroup`, `children` (single element)

#### ToolButton

Use `ToolButton` when a toolbar or icon button needs tips. Its `title` prop is consumed by the editor tooltip system and is not forwarded as a native DOM `title`.

```tsx
<IEditor.React.ToolButton
    className="toolbar-icon-button"
    title="Refresh assets"
    aria-label="Refresh assets"
    onClick={refresh}
>
    <IEditor.React.EditorImage src={refreshIcon} />
</IEditor.React.ToolButton>
```

A tool button without tips can use a normal `<button>`. Do not write `<button title="...">` when the title is intended as tool-button tips; use `ToolButton` instead. Use `TooltipTarget` for tips on non-button elements.

#### Popup

General-purpose anchored popup. Portals into the shadow root so it renders above all other content. Closes on outside click or Escape.

```tsx
<Popup
    open={open}
    onClose={() => setOpen(false)}
    className="select-input-popup"
    maxHeight={200}
    renderTrigger={({ ref }) => (
        <button ref={ref} onClick={() => setOpen(o => !o)}>Options ▾</button>
    )}
>
    <div style={{ padding: 8, display: "flex", flexDirection: "column", gap: 4 }}>
        <button onClick={handleA}>Action A</button>
        <button onClick={handleB}>Action B</button>
    </div>
</Popup>
```

Props: `open`, `onClose`, `onCancel`, `onOpen`, `renderTrigger`, `anchorRef`, `className`, `maxHeight`, `width`, `gap`, `children`

#### ResizeHandle

A draggable divider for building resizable panel layouts. Handles pointer capture, cross-iframe drag, keyboard arrow keys, and double-click reset. CSS classes `resize-handle` and `resize-handle-vertical` / `resize-handle-horizontal` are applied automatically; style with `cursor`, `className`, or `style` props.

```tsx
const [width, setWidth] = useState(200);

// Vertical handle (drags left/right to resize a column)
<div style={{ display: "flex" }}>
    <div style={{ width }}>Left panel</div>
    <ResizeHandle
        orientation="vertical"
        value={width}
        min={100}
        max={400}
        onResize={setWidth}
        onReset={() => setWidth(200)}
    />
    <div style={{ flex: 1 }}>Right panel</div>
</div>

// Horizontal handle (drags up/down)
<ResizeHandle
    orientation="horizontal"
    value={height}
    min={80}
    onResize={setHeight}
/>
```

Key props:

| Prop | Type | Description |
|---|---|---|
| `orientation` | `"vertical" \| "horizontal"` | Drag axis. Vertical = left/right arrows; horizontal = up/down arrows |
| `onResize` | `(value: number) => void` | Called with the new clamped value on every pointer move |
| `onResizeStart` | `(event) => { value, min?, max? } \| false \| void` | Override starting value/bounds; return `false` to cancel drag |
| `onReset` | `() => void` | Called on double-click or Enter key |
| `value` | `number` | Current size (passed to ARIA and used as keyboard baseline) |
| `min` / `max` | `number` | Clamp range |
| `keyboardStep` | `number` | Arrow key step size (default `10`) |
| `reverse` | `boolean` | Invert drag direction (useful for right/bottom anchored panels) |
| `disabled` | `boolean` | Disables drag and keyboard interaction |

---

### Graph Editors: `IEditor.Flow` and `IEditor.StateGraph`

The IDE provides two reusable React graph editors. Choose by graph semantics:

| Need | Runtime namespace | Type namespace | State model |
|---|---|---|---|
| Nodes with typed input/output ports, data-flow or logic links, comments, minimap, undo/redo | `IEditor.Flow` | `IEditor.IFlow` | `GraphStore` plus `IEditor.Flow.commands` |
| Pinless state-machine nodes with direct `sourceId -> targetId` transitions, arrows, self-loops, and fan-out | `IEditor.StateGraph` | `IEditor.IStateGraph` | Controlled `nodes`, `edges`, selection, and viewport props |

These are siblings of `IEditor.React`. Runtime components and helpers are under `IEditor.Flow` / `IEditor.StateGraph`; TypeScript interfaces are under `IEditor.IFlow` / `IEditor.IStateGraph`.

#### Port-based Node Graph (`IEditor.Flow`)

`IEditor.Flow` is the shared React Flow-based graph core used for port-based editors. Its serializable `GraphData` contains nodes, links, comments, and viewport state. A `GraphStore` is the single source of truth and includes subscriptions plus undo/redo.

Create the store and registries once, then render `GraphEditor`:

```tsx
const graphStore = IEditor.Flow.createGraphStore(initialGraph);
const nodeRegistry = new IEditor.Flow.NodeRegistry(nodeDefinitions);
const portTypes = new IEditor.Flow.PortTypeRegistry(portTypeDefinitions);

this._react = new IEditor.ReactDOM();
this._panel = this._react;
this._react.render(
    <IEditor.Flow.GraphEditor
        store={graphStore}
        registry={nodeRegistry}
        portTypes={portTypes}
        showMiniMap
    />
);
```

Use the type namespace when declaring graph data and extension points:

```tsx
const nodeDefinitions: IEditor.IFlow.NodeDefinition[] = [{
    typeId: "my-plugin.log",
    title: "Log",
    menuPath: "My Plugin/Log",
    create(position) {
        return {
            id: IEditor.Flow.nextId("node"),
            typeId: "my-plugin.log",
            position,
            inputs: [{
                id: "message",
                kind: "in",
                dataType: "string",
                label: "Message"
            }],
            outputs: [],
            data: {}
        };
    }
}];

const portTypeDefinitions: IEditor.IFlow.PortTypeInfo[] = [
    { key: "string", label: "String", color: "var(--bp-type-string)" }
];
```

Common runtime APIs:

- `GraphEditor`, `DefaultNodeBody`
- `createGraphStore()` / `GraphStore`
- `NodeRegistry`, `PortTypeRegistry`
- `emptyGraph()`, `nextId()`, `findNode()`, `linksOfPort()`
- `commands.addNode()`, `connect()`, `removeNodes()`, `moveNodes()`, `setNodeData()`, `setInputValue()`, comments, copy, and paste

`GraphEditor` can also customize connection validation, node body renderers, inline port inputs, canvas drop, context menus, selection callbacks, zoom limits, breakpoints, and animated debug-flow edges. Use `controllerRef` for imperative `focusNode()`.

Dispose the store along with the panel:

```ts
onDestroy() {
    graphStore.dispose();
    this._react?.dispose();
}
```

#### State-machine Graph (`IEditor.StateGraph`)

`IEditor.StateGraph` is for pinless state-machine diagrams. Nodes use `x`/`y` coordinates, and each edge directly names its `sourceId` and `targetId`. Unlike `IEditor.Flow`, it does not expose a graph store: the plugin owns the arrays and feeds updated state back through callbacks.

```tsx
type StateMachineViewProps = {
    nodes: IEditor.IStateGraph.StateGraphNode[];
    edges: IEditor.IStateGraph.StateGraphEdge[];
    setNodes: React.Dispatch<React.SetStateAction<IEditor.IStateGraph.StateGraphNode[]>>;
    addTransition(sourceId: string, targetId: string): void;
};

function StateMachineView({ nodes, edges, setNodes, addTransition }: StateMachineViewProps) {
    const [selectedNodes, setSelectedNodes] = useState<ReadonlySet<string>>(new Set());
    const [selectedEdges, setSelectedEdges] = useState<ReadonlySet<string>>(new Set());

    return (
        <IEditor.StateGraph.StateGraphEditor
            nodes={nodes}
            edges={edges}
            selectedNodeIds={selectedNodes}
            selectedEdgeIds={selectedEdges}
            showControls
            onSelectionChange={(nodeIds, edgeIds) => {
                setSelectedNodes(new Set(nodeIds));
                setSelectedEdges(new Set(edgeIds));
            }}
            onNodesMove={(ids, dx, dy) => {
                const moved = new Set(ids);
                setNodes(current => current.map(node =>
                    moved.has(node.id)
                        ? { ...node, x: node.x + dx, y: node.y + dy }
                        : node
                ));
            }}
            canConnect={(sourceId, targetId) => sourceId !== targetId}
            onConnectRejected={(sourceId, targetId) => {
                showConnectionError(sourceId, targetId);
            }}
            onConnect={addTransition}
        />
    );
}
```

Use `nodeStyles` to style node kinds, `defaultNodeId` to mark the default state, and callbacks for connect, move, double-click, context menus, deletion, selection, and viewport changes. Return `false` from `canConnect(sourceId, targetId)` to reject a transition; `onConnectRejected(sourceId, targetId)` then lets the host show feedback. It also receives `targetId: null` when `apiRef.beginLink(sourceId)` cannot find the source node. An `apiRef` exposes `focusNode()`, `fitView()`, `beginLink()`, `cancelLink()`, and `clientToGraph()`. For an initial viewport without mounting the component, use `IEditor.StateGraph.fitNodesToView()`; default dimensions are exported as `NODE_WIDTH` and `NODE_HEIGHT`.

---

### Timeline Editor (`IEditor.Timeline`)

`IEditor.Timeline` is the shared React Timeline for hierarchical tracks, keyframes, event markers, numeric curves, and host-neutral interval items such as clips. Runtime values are under `IEditor.Timeline`; use `IEditor.ITimeline` for all Timeline interfaces.

`TimelineEditor` reads and edits the supplied `TimelineDocument` directly. Keep the document object stable instead of rebuilding it every render. After the host mutates that object outside Timeline operations, change `dataVersion` so cached evaluation and drawing are refreshed. When the editor handle is available, prefer its `document` operations for normal key, track, evaluation, rename, and frame-rate work.

```tsx
function ClipTimeline() {
    const documentRef = useRef<IEditor.ITimeline.TimelineDocument>({
        fps: 30,
        totalFrame: 60,
        aniData: {
            name: "Root",
            prop: [{
                name: "opacity",
                label: "Opacity",
                keys: [
                    { f: 0, val: 0 },
                    { f: 30, val: 1 }
                ]
            }]
        }
    });
    const timelineRef = useRef<IEditor.ITimeline.TimelineEditorHandle | null>(null);
    const [dataVersion, setDataVersion] = useState(0);

    const actions = useMemo<IEditor.ITimeline.TimelineActions>(() => ({
        onCurrentFrameChange(frame) {
            previewAtFrame(frame);
        },
        onDataModified() {
            saveTimelineDocument(documentRef.current);
        },
        onKeySelectionChange(selection) {
            showSelectedKeys(selection);
        },
        onEditTransactionChange(active) {
            setEditing(active);
        }
    }), []);

    function addKeyFromHost() {
        documentRef.current.aniData?.prop?.[0].keys?.push({ f: 60, val: 0 });
        setDataVersion(version => version + 1);
    }

    return <div style={{ height: "100%", minHeight: 180, display: "flex", flexDirection: "column" }}>
        <button onClick={addKeyFromHost}>Add key</button>
        <div style={{ flex: 1, minHeight: 0 }}>
            <IEditor.Timeline.TimelineEditor
                ref={timelineRef}
                data={documentRef.current}
                dataVersion={dataVersion}
                actions={actions}
                mode="frame"
            />
        </div>
    </div>;
}
```

Give the Timeline's parent a real width and height because the canvas follows its container with `ResizeObserver`.

Keep these parts distinct:

- **Document model**: `TimelineDocument` owns `aniData`, `event`, and document capabilities. Its hierarchy uses `TimelineLayer`; layers contain `TimelineKey` and optional `TimelineRange` arrays. `TimelineEvent`, `TimelineValue`, `TimelineFrameData`, and `TimelineTweenInfo` cover the remaining common value shapes. Numeric values support Curve mode; strings and booleans are discrete. Opaque values can use `TimelineCustomValueAdapter`.
- **View state**: `TimelineViewState` stores scrolling, scale, playhead, expanded tracks, and selections separately from document data. Create an empty state with `IEditor.Timeline.createTimelineViewState()`, and persist it through `TimelineEditorHandle.getViewState()` / `setViewState()`.
- **Host actions and history**: `TimelineActions` supplies value lookup, edit/render/playback callbacks, selection notifications, context menus, range constraints, overlays, clipboard, drag/drop, and `onEditTransactionChange`. Pass the host's `IEditor.IDataHistory` through the `history` prop when Timeline edits should participate in undo/redo.
- **Editor state**: `getSnapshot()` returns a `TimelineEditorSnapshot` containing `position`, `mode`, `totalFrame`, `rowHeight`, `readOnly`, `modified`, `playing`, and `fps`. Use `setPosition()`, `setMode()`, and `setModified()` for the corresponding state changes.
- **Document handle**: `handle.document` is a `TimelineDocumentHandle` with `getData`, `getSaveData`, `getLayer`, `getKey`, `addKey`, `writeKey`, `removeTrack`, `getAdjacentKeys`, path rename helpers, `evaluate`, `evaluatePath`, `visitValues`, and `setFps`. `addKey` performs the normal history and rendering path; reserve `writeKey`, which does not force an immediate render, for a host-controlled batch.
- **Selection handle**: `handle.selection` is a `TimelineSelectionHandle` for querying, replacing, clearing, or removing selected tracks and selecting their keys.
- **Navigation and layout**: the main handle provides `focusPosition`, `focusRange`, `focusTrack`, `fitView`, `getTrackAt`, `getTrackLayout`, `setTrackScrollTop`, `setTrackOpen`, `hasTrackKeyAt`, `getTrackKeyColor`, `positionToTime`, playback, command, and coordinate-conversion methods.

Example handle operations:

```ts
const timeline = timelineRef.current;
if (timeline) {
    timeline.setPosition(12);
    timeline.document.addKey("Root::opacity", 12, 0.5);
    const value = timeline.document.evaluatePath("Root::opacity", 12);
    timeline.selection.setTracks(["Root::opacity"]);
    const snapshot = timeline.getSnapshot();
}
```

Important `TimelineEditorProps` beyond `data`, `actions`, `mode`, and `viewState` include `history`, `rowHeight`, `trackEndPadding`, `readOnly`, `recordMode`, `tweenEditable`, and `currentValueUpdate`. When `currentValueUpdate` is enabled, provide `TimelineActions.getCurrentValue`.

Canonical paths use `.` between hierarchy names and `::` before the property hierarchy. Encode every user-controlled name segment with `encodeTimelinePathSegment()` so literal dots remain unambiguous; use `decodeTimelinePathSegment()` for one returned segment and `getTimelineNamePath()` to obtain the hierarchy portion before `::`. Prefer semantic `executeCommand()` values such as `"copy"`, `"paste"`, `"deleteSelection"`, or `"togglePlayback"` instead of synthesizing keyboard events.

For interval editing, put `TimelineRange` objects on a layer's `ranges`. Each range needs a stable `id`, `start`, and `end`; `movable`, `trackMovable`, `trimStart`, `trimEnd`, `locked`, `interactive`, and `draw` control behavior. Keep host data in `payload`, and use `onRangeChanging` for previews plus `onRangeModified` for the committed edit.

---

### Built-in Theme Reference

ReactDOM automatically injects the active dark or light theme plus the editor component stylesheet into its Shadow DOM. Plugins can use the same semantic tokens, styled elements, and classes in both modes.

> **Rules:** Do not hard-code dark-theme colors. There are no built-in layout utility classes, so use inline layout styles or a small custom CSS file. For tokens not listed here, inspect the current `theme.css` and `theme-light.css`; they are the source of truth.

#### CSS Custom Properties

Override a variable only for an intentional plugin-specific skin, for example `:host { --accent: #e06c75; }`.

| Group | Tokens |
|---|---|
| Backgrounds | `--bg-darkest`, `--bg-dark`, `--bg-sunken`, `--bg-base`, `--bg-recessed`, `--bg-elevated`, `--bg-surface`, `--bg-hover`, `--bg-pressed`, `--bg-input`, `--bg-header`, `--bg-muted-surface`, `--bg-titlebar`, `--body-bg` |
| Accent and selection | `--accent`, `--accent-hover`, `--accent-muted`, `--accent-strong`, `--drop-indicator`, `--list-item-over`, `--list-item-selected`, `--list-item-selected-border`, `--list-item-selected-blur`, `--list-row-selected`, `--toggle-button-selected-bg`, `--toggle-button-selected-text` |
| Primary controls | `--primary`, `--primary-hover`, `--primary-text`, `--control-hover-subtle`, `--group-bg-muted`, `--group-accent-line` |
| Text | `--text`, `--text-bright`, `--text-muted`, `--text-disabled`, plus the `--hierarchy-title-*` state tokens |
| Panel chrome | `--panel-bg`, `--panel-topbar-bg`, `--panel-tab-bg`, `--panel-tab-bg-hover`, `--panel-tab-bg-active`, `--panel-tab-text`, `--panel-tab-text-hover`, `--panel-tab-text-active`, `--panel-tool-hover-bg`, `--inspector-tab-*`, `--play-controls-*` |
| Icons | `--ui-icon-filter`, `--ui-icon-opacity`, `--ui-icon-hover-opacity`, `--panel-tab-icon-opacity` |
| Borders and inputs | `--border`, `--border-subtle`, `--border-light`, `--border-hover`, `--checkbox-*`, `--select-popup-*`, `--select-option-*` |
| Status | `--status-warning`, `--status-warning-strong`, `--status-error`, `--status-error-bright`, `--status-close-bg` |
| Popup and tags | `--popup-menu-bg`, `--popup-menu-hover`, `--popup-menu-separator`, `--badge-bg`, `--badge-text`, `--tag-border`, `--tag-text`, `--tag-bg` |
| Drag and drop | `--drag-row-bg`, `--drag-row-bg-soft`, `--drag-row-outline`, `--drop-line-color`, `--drop-line-shadow-inner`, `--drop-line-glow` |
| Overlays and shadows | `--hud-*`, `--shadow-color-subtle`, `--shadow-color-medium`, `--shadow-color-strong`, `--row-zebra`, `--progress-shine`, `--resize-handle` |
| Progress and markers | `--progress-track`, `--progress-border`, `--progress-fill`, `--marker-fill`, `--marker-shadow`, `--marker-glyph`, `--round-expand-*` |
| Geometry and type | `--radius`, `--radius-lg`, `--radius-sm`, `--font-size`, `--font-size-small`, `--font-size-tiny`, `--font-family`, `--transition` |
| Scrollbars | `--scrollbar-size`, `--scrollbar-thumb`, `--scrollbar-track`, `--scrollbar-thumb-soft`, `--scrollbar-thumb-soft-hover`, `--scrollbar-thumb-firefox` |
| Graph and code | `--grid-minor`, `--grid-major`, `--bp-type-*`, `--flow-*`, `--curve-*`, `--animator-controller-*`, `--hljs-*` |
| Timeline canvas | `--timeline-property-row-bg`, `--timeline-property-row-alt`, `--timeline-track-bg`, `--timeline-curve-bg`, `--timeline-track-row-source-bg`, `--timeline-track-line`, `--timeline-track-row-line`, `--timeline-track-grid-line`, `--timeline-track-grid-line-major`, `--timeline-record-ruler-bg`, `--timeline-record-ruler-text`, `--timeline-selection-bg`, `--timeline-select-keys-bg`, `--timeline-key-color`, `--timeline-key-dir-color`, `--timeline-key-selected-color`, `--timeline-clip-bg-start`, `--timeline-clip-bg-end`, `--timeline-clip-text`, `--timeline-clip-selected-border`, `--timeline-property-key-state-border`, `--timeline-overlay-bg`, `--timeline-tween-grid-line`, `--timeline-tween-path`, `--timeline-path-color-saturation`, `--timeline-path-color-lightness` |

Use `--toggle-button-selected-bg` and `--toggle-button-selected-text` for persistent on/off state represented by `aria-pressed="true"`; the built-in `.toolbar-icon-button` already consumes them. Timeline clip colors use the `--timeline-clip-*` tokens.

#### Standalone icons across dark and light themes

When the same standalone icon must support both themes, use `--ui-icon-filter`. The light theme sets it to darken icons authored for dark surfaces, while the dark theme sets it to `none`.

```css
.my-monochrome-icon {
    width: 16px;
    height: 16px;
    filter: var(--ui-icon-filter);
    opacity: var(--ui-icon-opacity);
}

.my-tool:hover .my-monochrome-icon {
    opacity: var(--ui-icon-hover-opacity);
}
```

Built-in selectors such as `.toolbar-icon-button > .image .image-content`, `button.icon > .image .image-content`, and `.search-input > .image .image-content` already apply the filter. A standalone `EditorImage` does not; add a custom class only when that icon needs to adapt across dark and light themes.

#### Auto-styled HTML Elements

These elements are styled automatically — just use the raw HTML tag:

- `<button>` — themed surface with hover/active/disabled states
- `<input type="text|number|search|password|url|email">` — themed input with focus ring
- `<textarea>` — multi-line input, resizable
- `<input type="checkbox">` — custom styled checkbox, accent color when checked
- `<select>` — custom dropdown arrow
- `<table>`, `<th>`, `<td>` — styled table
- `<a>` — accent colored link

#### Component Classes

| Class | Description |
|---|---|
| `button.primary` | Primary CTA button (`--primary` blue background) |
| `button.icon` / `.btn-icon` | Small 22×22 icon button (transparent bg, no border by default) |
| `.toolbar-icon-button` | 24×24 toolbar icon button; set `aria-pressed="true"` for persistent state using the `--toggle-button-selected-*` tokens |
| `.tab` | Tab button; add `.active` or `aria-selected="true"` for the selected tab |
| `.search-input` | Search bar container — wrap an icon element + `<input>` inside |
| `.select-input` | Custom select-like trigger button (IDE select widget style) |
| `.editor-slider` | Styled `<input type="range">` (custom track + thumb) |
| `.numeric-input` | Drag-to-edit number input container |
| `.text-muted` | Apply `var(--text-muted)` color |
| `.editor-list` / `.editor-list-row` | List container + rows; add `.is-selected` or `.selected` on rows for selection highlight |
| `.progress` / `.progress-fill` | Progress bar track and fill; add `.progress-animated` on `.progress` for animated shimmer |
| `.ide-tooltip-content` | Tooltip popup bubble (max-width 300px) |

#### Example: Using Built-in Styles

```tsx
function SettingsPanel() {
    const [tab, setTab] = React.useState("general");
    return (
        <div style={{ display: "flex", flexDirection: "column", height: "100%", overflow: "hidden" }}>
            {/* Tabs */}
            <div style={{ display: "flex", borderBottom: "1px solid var(--border)" }}>
                <button className={`tab${tab === "general" ? " active" : ""}`}
                    onClick={() => setTab("general")}>General</button>
                <button className={`tab${tab === "advanced" ? " active" : ""}`}
                    onClick={() => setTab("advanced")}>Advanced</button>
            </div>

            {/* Content — raw elements auto-styled */}
            <div style={{ flex: 1, overflow: "auto", padding: 8, display: "flex", flexDirection: "column", gap: 8 }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                    <label style={{ minWidth: 80, color: "var(--text-muted)" }}>Name</label>
                    <input type="text" placeholder="Enter name..." />
                </div>
                <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
                    <input type="checkbox" id="enabled" />
                    <label htmlFor="enabled">Enable feature</label>
                </div>
            </div>

            {/* Footer */}
            <div style={{ display: "flex", gap: 6, justifyContent: "flex-end",
                          padding: "8px 10px", borderTop: "1px solid var(--border)" }}>
                <button>Cancel</button>
                <button className="primary">Save</button>
            </div>
        </div>
    );
}
```

#### Overriding Theme Variables

```css
/* MyPlugin.css — override accent color and input radius */
:host {
    --accent: #e06c75;
    --accent-hover: #e88992;
    --radius-lg: 3px;
}

/* Additional custom styles */
.my-custom-card {
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 12px;
}
```

---

## 3. Menu Plugin

**Process**: UI

### Static menu item

```typescript
class MenuExt {
    @IEditor.menu("App/tool/MyCommand", {
        label: "My Command",
        accelerator: "ctrl+shift+m",
    })
    static onMyCommand() {
        console.log("menu clicked");
    }
}
```

### Menu with conditions

```typescript
class MenuExt {
    @IEditor.menu("Hierarchy/test", {
        position: "before openDevTools",
        enableTest: () => Editor.scene?.getSelection().length > 0
    })
    static test() {
        // Only enabled when nodes are selected
    }
}
```

### Dynamic menu at runtime

```typescript
@IEditor.onLoad
static onLoad() {
    Editor.extensionManager.addMenuItem("App/tool/DynamicMenu", () => {
        console.log("dynamic menu clicked");
    }, { label: "Dynamic Menu", id: "dynamicMenu" });
}
```

### Popup context menu

Create a popup menu once and reuse it. Do not call anonymous `Menu.create([...])` from a click, pointer, or `contextmenu` handler; that creates a new registered menu for every interaction.

```typescript
const CONTEXT_MENU_ID = "MyCompany.MyPlugin.ItemContextMenu";

type ContextMenuData = {
    targetId: string;
};

let contextMenu: IEditor.IMenu | undefined;

function getContextMenu(): IEditor.IMenu {
    if (contextMenu)
        return contextMenu;

    contextMenu = IEditor.Menu.getById(CONTEXT_MENU_ID)
        ?? IEditor.Menu.create(CONTEXT_MENU_ID, [
            {
                id: "open",
                label: "Open",
                click: (_itemId, data: ContextMenuData) => openItem(data.targetId)
            },
            {
                id: "delete",
                label: "Delete",
                click: (_itemId, data: ContextMenuData) => deleteItem(data.targetId)
            }
        ]);
    return contextMenu;
}

function showContextMenu(event: MouseEvent, targetId: string, canDelete: boolean) {
    event.preventDefault();

    const menu = getContextMenu();
    menu.setItemEnabled("delete", canDelete);
    menu.show(
        undefined,
        { x: event.clientX, y: event.clientY },
        { targetId } satisfies ContextMenuData
    );
}
```

Menu lifecycle rules:

- Use a globally unique, plugin-prefixed menu ID. Menu IDs share one registry.
- Prefer creating the menu during plugin/panel initialization and caching the instance. Lazy creation is also safe when it uses the getter pattern above.
- `IEditor.Menu.create(sameId, ...)` does not return the existing menu; it throws. Always call `getById()` before an ID-based lazy create.
- Give actionable menu items explicit, stable IDs so the reused menu can update them.
- For changing content or state, reuse the menu and call `setItems()`, `setItemEnabled()`, `setItemVisible()`, `setItemChecked()`, or `setItemLabel()` before `show()`.
- When multiple live component instances need callbacks bound to different owners, cache one menu per owner and give each menu a unique instance ID; do not let instances accidentally share callbacks.

---

## 4. Dialog

**Process**: UI

```tsx
import styles from "./MyDialog.css";

function MyDialogView() {
    return <div style={{ padding: 12 }}>Dialog content</div>;
}

export class MyDialog extends IEditor.Dialog<IEditor.ReactDOM> {
    async create() {
        this.contentPane = new IEditor.ReactDOM();
        this.contentPane.setSize(480, 320);
        this.contentPane.adoptStyles(styles);
        this.contentPane.render(<MyDialogView />);
        this.title = "My Dialog";
        this.resizable = true;
    }

    onShown() { /* dialog visible */ }
    onHide() { /* dialog hidden */ }

    dispose() {
        this.contentPane?.dispose();
        super.dispose();
    }
}

// Show dialog
Editor.showDialog(MyDialog, null);
```

### Dialog with InspectorPanel

```tsx
export class ConfigDialog extends IEditor.Dialog<IEditor.ReactDOM> {
    private _data: any;
    private _model: InstanceType<typeof IEditor.React.InspectorPanelModel>;

    async create() {
        this._data = IEditor.DataWatcher.watch({ name: "default", value: 100 });
        this._model = new IEditor.React.InspectorPanelModel();
        this._model.allowUndo = true;
        this._model.inspect(this._data, {
            name: "MyCompany.MyPlugin.ConfigType",
            properties: [
                { name: "name", type: String },
                { name: "value", type: Number }
            ]
        });

        this.contentPane = new IEditor.ReactDOM();
        this.contentPane.setSize(500, 400);
        this.contentPane.render(<IEditor.React.InspectorPanel model={this._model} />);
        this.title = "Configuration";
        this.resizable = true;
    }

    dispose() {
        this._model?.resetInspectors();
        this.contentPane?.dispose();
        super.dispose();
    }
}
```

---

## 5. Custom Inspector Field

**Process**: UI  
**Use**: Custom property editor in Inspector panel

React inspector fields initialize metadata in `create()` and return JSX from `render()`. Prefer the existing built-in field types and `IEditor.React` controls before registering a custom field.

```tsx
@IEditor.inspectorField("MyCompany.MyPlugin.ActionField")
export class ActionField extends IEditor.PropertyField {
    create(): IEditor.IPropertyFieldCreateResult {
        return {
            stretchWidth: true,
            captionDisplay: "hidden",
        };
    }

    render(): React.ReactNode {
        return (
            <IEditor.React.ToolButton
                title={this.property.tips || "Execute"}
                disabled={this.react?.readonly}
                onClick={() => this.target.setValue(Date.now())}
            >
                Execute
            </IEditor.React.ToolButton>
        );
    }
}

// @property({ type: Number, inspector: "MyCompany.MyPlugin.ActionField" })
// lastRun: number;
```

---

## 6. Inspector Layout

**Process**: UI  
**Use**: Custom Inspector for asset files

### MetaData layout (import settings for raw assets)

```typescript
@IEditor.regClass()
export class ABCImportSettings {
    @IEditor.property(String)
    name: string = "";
}

@IEditor.inspectorLayout("asset")
export class ABCInspectorLayout extends IEditor.MetaDataInspectorLayout {
    constructor() {
        super(ABCImportSettings);
    }

    accept(asset: IEditor.IAssetInfo): boolean {
        return asset.ext === "abc";
    }
}
```

### Resource layout (live read/write assets)

```typescript
@IEditor.inspectorLayout("asset")
export class ABCResourceLayout extends IEditor.ResourceInspectorLayout {
    accept(asset: IEditor.IAssetInfo): boolean {
        return asset.ext === "abc";
    }
}
```

### File layout (offline config-type assets)

```typescript
@IEditor.regClass()
export class ABCFileType {
    @IEditor.property(String)
    name: string = "";
}

@IEditor.inspectorLayout("asset")
export class ABCFileLayout extends IEditor.FileInspectorLayout {
    constructor() {
        super(ABCFileType);
    }

    accept(asset: IEditor.IAssetInfo): boolean {
        return asset.ext === "abc";
    }
}
```

---

## 7. Settings & Preferences

**Process**: UI (read/write), Scene (read-only after sync)

### Create settings

```typescript
@IEditor.regClass()
export class MyPluginSettings {
    @property({ type: Boolean, default: true })
    enableFeature: boolean = true;

    @property(String)
    apiEndpoint: string = "";

    @property({ type: Number, min: 1, max: 60 })
    timeout: number = 30;
}

class PluginMain {
    @IEditor.onLoad
    static onLoad() {
        Editor.extensionManager.createSettings(
            "MyPluginSettings",
            "project",    // "project"|"local"|"application"|"memory"
            MyPluginSettings
        );
    }
}
```

### Access settings

```typescript
// UI process (read/write)
let settings = Editor.getSettings("MyPluginSettings");
settings.data.apiEndpoint = "https://example.com";

// Scene process (read-only, must sync first)
let settings = EditorEnv.getSettings("MyPluginSettings");
await settings.sync();
console.log(settings.data.apiEndpoint);
```

### Settings panel in Preferences/Project Settings

```tsx
@IEditor.panel("MyCompany.MyPlugin.Preferences", {
    usage: "preference",       // "preference"|"project-settings"|"build-settings"
    title: "My Plugin"
})
export class MyPluginPrefs extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;
    private _model: InstanceType<typeof IEditor.React.InspectorPanelModel>;

    async create() {
        this._model = new IEditor.React.InspectorPanelModel();
        this._model.inspect(
            Editor.getSettings("MyPluginSettings").data,
            "MyPluginSettings"
        );
        this._react = new IEditor.ReactDOM();
        this._panel = this._react;
        this._react.render(<IEditor.React.InspectorPanel model={this._model} />);
    }

    onDestroy() {
        this._model?.resetInspectors();
        this._react?.dispose();
    }
}
```

### Runtime-accessible settings

```typescript
Editor.extensionManager.createSettings("MyPluginSettings",
    { location: "project", contributeToPlayerConfig: true });

// At runtime:
console.log(Laya.PlayerConfig["MyPluginSettings"]);
```

### API keys and authenticated HTTP: `IEditor.SecretStorage`

**Process**: UI API; secret storage and authenticated transport run in the host/main process. Do not assume an `EditorEnv.SecretStorage` or Preview equivalent.

Use `IEditor.SecretStorage` for plugin/user API keys instead of serializing them into ordinary settings, project assets, or PlayerConfig. It exposes only `set(name, value, options): Promise<boolean>`, `has(name): Promise<boolean>`, `remove(name): Promise<void>`, and `request<T>(name, url, options?): Promise<{ status, headers, data: T }>`. There is no plaintext getter: save a user-entered key once, then request by name without copying the credential into request headers, query parameters, or bodies yourself.

```ts
const secretName = "com.example.my-plugin.api";

// Call when the user saves or replaces their key, not on every request.
async function saveApiKey(userEnteredKey: string): Promise<boolean> {
    return IEditor.SecretStorage.set(secretName, userEnteredKey, {
        baseURL: "https://api.example.com/v1/",
        allowedOrigins: ["https://upload.example.com"], // Optional extra origin.
        // auth defaults to { type: "bearer" }.
    });
}

async function generate(prompt: string, signal?: AbortSignal) {
    if (!await IEditor.SecretStorage.has(secretName))
        throw new Error("Configure an API key first.");

    const response = await IEditor.SecretStorage.request<{ jobId: string }>(
        secretName, "generate", {
            method: "POST",
            body: { prompt }, // JSON by default.
            signal,
        }
    );
    if (response.status < 200 || response.status >= 300)
        throw new Error(`Service returned HTTP ${response.status}`);
    return response.data; // Full provider body; not automatically unwrapped.
}

// Call when the user explicitly removes the saved key.
async function removeApiKey(): Promise<void> {
    await IEditor.SecretStorage.remove(secretName);
}
```

Check the boolean returned by `saveApiKey`: `true` means encrypted persistence succeeded; `false` means the key is usable only in memory (including CLI). Tell the user when a key will not survive restart. Storage failures can reject; do not silently fall back to plaintext settings. Clear the key-entry UI after saving and do not log the entered value.

Names are editor-wide, not project-scoped or a plugin permission boundary. Use a company/plugin/purpose prefix to avoid collisions; other plugins in the same editor can use or modify ordinary named secrets. The built-in `com.layaair.aigc` name is host-managed: use `has` and `request` with the intended platform API path, but do not `set` or `remove` it. Availability may be acquired lazily by the host; do not hard-code platform credentials or service addresses.

**Authentication and target origins** are fixed when saving the key:

| `auth` | Host injection |
| --- | --- |
| Omitted or `{ type: "bearer" }` | `Authorization: Bearer <secret>` |
| `{ type: "header", name: "x-goog-api-key" }` | Named header; optionally add a `prefix` string |
| `{ type: "query", name: "key" }` | Named query parameter |
| `{ type: "body", name: "api_key" }` | Named field in a JSON object, form, or multipart body |

`baseURL` must be an HTTP(S) URL without credentials, query, or fragment; prefer HTTPS for remote services. Relative targets resolve against it. Absolute URLs are accepted only on its origin or an origin listed in `allowedOrigins`. Extra entries are origins (scheme, host, optional port), not paths or wildcards. Requests cannot expand this allowlist or override the injected authentication, and redirects are disabled. Cross-origin signed upload/download URLs that do not require this API key should be used directly without attaching the original key.

**Request and response handling**:

- `headers` carries extra provider headers; `query` accepts scalar values or arrays for repeated keys. Model parameters, task polling, and provider business codes remain plugin responsibilities.
- `bodyType` defaults to `"json"`; also supports `"text"` (string), `"binary"` (`Uint8Array`), `"form"` (record), and `"multipart"` (parts array). A multipart part is `{ name, value: string }`, `{ name, data: Uint8Array, filename, contentType? }`, or `{ name, filePath, filename?, contentType? }`; names may repeat. Resolve editor resource locators to absolute local paths before using `filePath`. Do not set a multipart boundary yourself.
- `responseType` defaults to `"json"`; alternatives are `"text"`, `"arrayBuffer"`, and `"stream"`. The result is `{ status, headers, data }`, not a Fetch `Response`: inspect `status`, do not use `.ok` or `.json()`. Header names are lowercase; the host redacts the used secret from returned content.
- HTTP non-2xx responses are returned for the plugin to handle; transport/decoding errors reject. No automatic retries, business-code interpretation, or task polling are performed.
- For `responseType: "stream"`, use `request<ReadableStream<Uint8Array>>(...)` and consume or cancel `response.data`; decode SSE/NDJSON framing in the plugin. Pass `signal: AbortSignal` for cancellation and clean up on panel/dialog disposal. Cancellation stops transport, not a remote job; cancel that through the provider's API when needed.
- `timeoutMs` defaults to 10 minutes and covers stream consumption; `maxResponseBytes` defaults to 128 MiB, including streamed data. Always consume or cancel streams rather than leaving them open.

---

## 8. Build Plugin

**Process**: Scene  
**Use**: Extend the build pipeline

```typescript
@IEditorEnv.regBuildPlugin("web")  // Platform: "web"|"*" for all
export class MyBuildPlugin implements IEditorEnv.IBuildPlugin {
    async onSetup(task: IEditorEnv.IBuildTask) {
        // Initial setup
    }

    async onStart(task: IEditorEnv.IBuildTask) {
        task.logger.debug("Build started");
    }

    async onCollectAssets(task: IEditorEnv.IBuildTask, assets: Set<IAssetInfo>) {
        // Add/remove assets from build
    }

    async onBeforeExportAssets(task: IEditorEnv.IBuildTask,
        exportInfoMap: Map<IAssetInfo, IAssetExportInfo>) {
        // Modify export settings before assets are exported
    }

    async onExportScripts(task: IEditorEnv.IBuildTask) {
        // Custom script export logic
    }

    async onAfterExportAssets(task: IEditorEnv.IBuildTask,
        exportInfoMap: Map<IAssetInfo, IAssetExportInfo>) {
        // Post-process exported assets
    }

    async onCreateManifest(task: IEditorEnv.IBuildTask) {
        // Modify asset manifest
    }

    async onCreatePackage(task: IEditorEnv.IBuildTask) {
        // Package creation (e.g., mini-game packages)
    }

    async onEnd(task: IEditorEnv.IBuildTask) {
        task.logger.debug("Build complete");
    }
}

// With priority (lower = earlier)
@IEditorEnv.regBuildPlugin("web", 10)
export class EarlyBuildPlugin implements IEditorEnv.IBuildPlugin { }
```

### Build utility methods

```typescript
task.logger.debug("message");
task.mergeConfigFile(path);
await IEditorEnv.utils.renderTemplateFile(templatePath, data);
await IEditorEnv.utils.installCli("@some/cli", options);
await IEditorEnv.utils.exeCli("npm", ["install"], options);
await IEditorEnv.utils.exec("command", args);
await IEditorEnv.utils.downloadFile(url, savePath);
```

---

## 9. Custom Build Target

**Process**: UI + Scene

```tsx
// UI process: register target
class PluginMain {
    @IEditor.onLoad
    static onLoad() {
        Editor.extensionManager.createSettings("MyPlatformSettings", "project");
        Editor.extensionManager.createBuildTarget("myplatform", {
            caption: "My Platform",
            icon: "editorResources/my-plugin/platform-icon.svg",
            settingsName: "MyPlatformSettings",
            inspector: "MyPlatformBuildSettings",
        });
    }
}

// UI process: build settings panel
@IEditor.panel("MyCompany.MyPlugin.PlatformBuildSettings", {
    usage: "build-settings",
    title: "My Platform"
})
export class MyPlatformBuildSettings extends IEditor.EditorPanel {
    private _react: IEditor.ReactDOM;
    private _model: InstanceType<typeof IEditor.React.InspectorPanelModel>;

    async create() {
        this._model = new IEditor.React.InspectorPanelModel();
        this._model.inspect(
            Editor.getSettings("MyPlatformSettings").data,
            "MyPlatformSettings"
        );
        this._react = new IEditor.ReactDOM();
        this._panel = this._react;
        this._react.render(<IEditor.React.InspectorPanel model={this._model} />);
    }

    onDestroy() {
        this._model?.resetInspectors();
        this._react?.dispose();
    }
}

// Scene process: build plugin
@IEditorEnv.regBuildPlugin("myplatform")
export class MyPlatformBuild implements IEditorEnv.IBuildPlugin {
    async onCreatePackage(task: IEditorEnv.IBuildTask) {
        task.config.runHandler = { serveRootPath: "" };
    }
}
```

### Trigger build programmatically

```typescript
// UI process
IEditor.BuildTask.start("web");

// Scene process
IEditorEnv.BuildTask.start("web");
```

---

## 10. Custom Asset Type

**Process**: UI

### Register file type, icon, and actions

```typescript
class PluginMain {
    @IEditor.onLoad
    static onLoad() {
        // Set icon
        Editor.extensionManager.setFileIcon(["abc"], "editorResources/my-plugin/abc.svg");

        // Set category (for filtering in asset browser)
        Editor.extensionManager.setFileType(["abc"], "ABC Files");

        // Set file actions
        Editor.extensionManager.addFileActions(["abc"], {
            onOpen: async (asset) =>
                IEditor.utils.openCodeEditor(Editor.assetDb.getFullPath(asset)),
            onCreateNode: async (asset) =>
                Editor.scene.createNode("Sprite", { /* props */ }),
            onDropToScene: async (asset) => true,
            onCreateInField: async (asset) => { /* handle drop to field */ }
        });
    }
}
```

---

## 11. Asset Import/Export

**Process**: Scene

### Asset Importer

```typescript
@IEditorEnv.regAssetImporter(["abc"])
export class ABCImporter extends IEditorEnv.AssetImporter {
    async handleImport(): Promise<any> {
        // Pre-process logic on import
        let data = await IEditorEnv.utils.readJsonAsync(this.fullPath);
        // Transform data...
        return data;
    }
}
```

### Asset Exporter

```typescript
@IEditorEnv.regAssetExporter(["abc"])
export class ABCExporter extends IEditorEnv.AssetExporter {
    async handleExport(): Promise<void> {
        // Set dependencies
        const links = [
            { obj: "data", prop: "url", url: "asset-uuid-here" }
        ];
        this.exportInfo.deps = this.parseLinks(links);

        // Modify output content
        this.exportInfo.contents[0] = {
            type: "text",
            data: JSON.stringify(transformedData)
        };
    }
}

// Exclude asset from build entirely
@IEditorEnv.regAssetExporter(["abc"], { exclude: true })
export class ABCExcluder extends IEditorEnv.AssetExporter { }
```

### Asset Saver (for live-editable resources)

```typescript
@IEditorEnv.regAssetSaver(["abc"])
export class ABCSaver implements IEditorEnv.IAssetSaver {
    async onSave(asset: IEditorEnv.IAssetInfo, res: ABCResource) {
        let data = IEditorEnv.SerializeUtil.encodeObj(res, null, { writeType: false });
        await IEditorEnv.utils.writeJsonAsync(
            EditorEnv.assetMgr.getFullPath(asset), data
        );
    }
}
```

### Asset Loader (runtime)

```typescript
@Laya.regClass()
export class ABCResource extends Laya.Resource {
    @Laya.property(String)
    name: string = "";
}

@Laya.regLoader(["abc"], null, true)
export class ABCLoader implements Laya.IResourceLoader {
    async load(task: Laya.ILoadTask): Promise<any> {
        let json = await task.loader.fetch(task.url, "json");
        let res = task.obsoluteInst ? task.obsoluteInst : new ABCResource();
        Object.assign(res, json);
        return res;
    }
}
```

---

## 12. Asset Thumbnail & Preview

**Process**: Scene (thumbnail gen) + UI (preview panel)

### Thumbnail

```typescript
// UI process: register
@IEditor.onLoad
static onLoad() {
    Editor.extensionManager.setFileThumbnail(["abc"], "ABCThumbnailGen");
}

// Scene process: generator
@IEditorEnv.regClass()
export class ABCThumbnailGen extends IEditorEnv.AssetThumbnail {
    async generate(asset: IEditorEnv.IAssetInfo): Promise<string | Buffer> {
        // Return file path or Buffer of the thumbnail image
    }
}
```

### Preview Panel

```tsx
// UI process
@IEditor.panel("MyCompany.MyPlugin.ABCPreview", { usage: "preview" })
export class ABCPreview extends IEditor.EditorPanel implements IEditor.IPreviewPanel {
    private _react: IEditor.ReactDOM;

    async create() {
        this._react = new IEditor.ReactDOM();
        this._panel = this._react;
        this._react.render(<div className="text-muted">Select an ABC asset</div>);
    }

    accept(asset: IEditor.IAssetInfo): boolean {
        return asset.ext === "abc";
    }

    async refresh(asset: IEditor.IAssetInfo, render3DCanvas: IEditor.IRender3DCanvas) {
        return render3DCanvas.createObject("ABCPreviewScript", "setAssetById", asset.id);
    }

    onDestroy() {
        this._react?.dispose();
    }
}

// Scene process
@IEditorEnv.regClass()
export class ABCPreviewScript extends IEditorEnv.AssetPreview {
    async setAsset(asset: IEditorEnv.IAssetInfo) {
        this.renderTarget = this.sprite;
    }
}
```

---

## 13. Asset Processor

**Process**: Scene  
**Use**: Pre/post-process assets on import

```typescript
@IEditorEnv.regAssetProcessor()
export class MyAssetProcessor implements IEditorEnv.IAssetProcessor {
    onPreprocessImage(assetImporter: IEditorEnv.IImageAssetImporter) {
        // Auto-set compression for non-sprite images
        if (assetImporter.config.textureType != 2) {
            assetImporter.config.platformDefault = { format: 10 };
        }
    }

    async onPostprocessAsset(assetImporter: IEditorEnv.IAssetImporter) {
        // Post-process any asset after import
    }
}
```

---

## 14. Scene Hook

**Process**: Scene  
**Use**: React to scene events

```typescript
@IEditorEnv.regSceneHook()
export class MySceneHook implements IEditorEnv.ISceneHook {
    onLoadScene() {
        // Scene loaded
    }

    onSaveScene(scene: IEditorEnv.IGameScene, data: any) {
        // Before scene save - can modify data
    }

    onCreateNode(scene: IEditorEnv.IGameScene, node: Laya.Node) {
        // Node created in editor
        // Example: auto-set anchor for UI sprites
        if (node instanceof Laya.Sprite) {
            node.anchorX = node.anchorY = 0.5;
        }
    }

    onCreateComponent(scene: IEditorEnv.IGameScene, comp: Laya.Component) {
        // Component added to node
    }
}
```

---

## 15. Custom Editor (Gizmos)

**Process**: Scene  
**Use**: Draw gizmos/handles for a specific component

### 3D Gizmos

```typescript
@IEditorEnv.customEditor(MyComponent)
export class MyComponentEditor extends IEditorEnv.CustomEditor {
    onSceneGUI(): void {
        // Interactive handles (called during gizmo phase)
        IEditorEnv.Handles.drawHemiSphere(this.owner.transform.position, 2);
    }

    onDrawGizmos(): void {
        // Always visible gizmos
        IEditorEnv.Gizmos.drawIcon(
            this.owner.transform.position,
            "editorResources/my-plugin/icon.png"
        );
    }

    onDrawGizmosSelected(): void {
        // Only visible when selected
        IEditorEnv.Gizmos.drawWireSphere(
            this.owner.transform.position, 5
        );
    }
}
```

### 2D Gizmos

```typescript
@IEditorEnv.customEditor(My2DComponent)
export class My2DEditor extends IEditorEnv.CustomEditor {
    private _circle: IEditorEnv.IGizmoCircle;

    onDrawGizmosSelected(): void {
        if (!this._circle) {
            let manager = IEditorEnv.Gizmos2D.getManager(this.owner);
            this._circle = manager.createCircle(10);
            this._circle.fill("#ff0");
        }
        this._circle.setLocalPos(10, 10);
    }
}
```

---

## 16. Asset Database API

**Use**: Query assets, convert paths, listen for changes. One of the most commonly used APIs in plugin development.

### IAssetInfo Properties

Asset info object available in both processes:

```typescript
interface IAssetInfo {
    id: string;               // Asset UUID
    name: string;             // Asset name (without extension)
    fileName: string;         // Full filename (with extension)
    file: string;             // Path relative to project root
    ext: string;              // File extension (without dot)
    type: AssetType;          // Asset type enum
    subType: string;          // Sub-type (e.g., "Spine" for spine JSON)
    parentId: string;         // Parent folder UUID
    hasChild: boolean;        // Whether asset has children
    children: ReadonlyArray<IAssetInfo>;
    ver: number;              // Version number (increments on change)
    flags: number;            // AssetFlags bitfield
    scriptType: AssetScriptType; // Script classification
}
```

### UI Process: Editor.assetDb

```typescript
// === Query Assets ===

// Get asset by ID or path (async)
let asset = await Editor.assetDb.getAsset("assets/textures/hero.png");
let asset = await Editor.assetDb.getAsset("uuid-string");

// Get asset synchronously (cached only)
let asset = Editor.assetDb.getAssetSync("assets/textures/hero.png");

// Search assets (by keyword + type filter)
let results = await Editor.assetDb.search("hero", [AssetType.Image]);

// Get folder contents
let children = await Editor.assetDb.getFolderContent(folderAsset.id, [AssetType.Image]);

// Check if asset matches type
let isImage = await Editor.assetDb.matchType(asset.id, [AssetType.Image]);

// === Path Conversion ===

// Get absolute path of asset
let fullPath = Editor.assetDb.getFullPath(asset);

// Relative path -> absolute path
let fullPath = Editor.assetDb.toFullPath("assets/textures/hero.png");

// Absolute path -> relative path
let relPath = Editor.assetDb.toRelativePath("/Users/me/project/assets/textures/hero.png");

// Get asset URL (file:///... format)
let url = Editor.assetDb.getURL(asset);

// Relative path -> URL
let url = Editor.assetDb.toURL("assets/textures/hero.png");

// Resolve a relative editor-resource locator; the second argument must be true
let resourceAsset = await Editor.assetDb.getAsset(
    "editorResources/my-plugin/locales",
    true
);

// Node.js IO requires the absolute filesystem path
let fullPath = Editor.assetDb.getFullPath(resourceAsset);

// === File Operations ===

// Create file
let newAsset = await Editor.assetDb.writeFile("assets/data/config.json", jsonContent);

// Create from template
let newAsset = await Editor.assetDb.createFileFromTemplate(
    "assets/scripts/MyScript.ts", "Script", { className: "MyScript" }
);

// Create folder
let folder = await Editor.assetDb.createFolder("assets/textures/characters");

// Rename (returns 0=success, -1=not found, 1=already exists, 2=IO error)
let result = await Editor.assetDb.rename(asset.id, "newName");

// Move assets
await Editor.assetDb.move([asset1.id, asset2.id], targetFolderId);

// Copy assets
await Editor.assetDb.copy([asset.id], targetFolderId);

// Delete assets
await Editor.assetDb.delete([asset]);

// === Metadata ===
await Editor.assetDb.setMetaData(asset.id, { quality: 0.8 });

// === Trigger reimport ===
Editor.assetDb.reimport([asset]);

// === Listen for asset changes ===
// IMPORTANT: You MUST remove the listener in @IEditor.onUnload to avoid leaks on plugin reload
function onAssetChanged(assetId: string, assetPath: string, assetType: AssetType, flag: AssetChangedFlag) {
    // flag: AssetChangedFlag.New / Modified / Deleted / Moved
    console.log(`Asset ${assetPath} changed: ${flag}`);
}
Editor.assetDb.onAssetChanged.add(onAssetChanged);

// In @IEditor.onUnload:
// Editor.assetDb.onAssetChanged.remove(onAssetChanged);

// === Wait for pending changes ===
await Editor.assetDb.flushChanges();
```

### Scene Process: EditorEnv.assetMgr

```typescript
// === Query Assets (synchronous) ===

// Get by ID or path
let asset = EditorEnv.assetMgr.getAsset("assets/textures/hero.png");
let asset = EditorEnv.assetMgr.getAsset("uuid-string");

// Get all assets (dictionary keyed by UUID)
let all = EditorEnv.assetMgr.allAssets;

// Search
let results = EditorEnv.assetMgr.findAssets("hero", [AssetType.Image]);

// Get all assets of a type
let images = EditorEnv.assetMgr.getAssetsByType([AssetType.Image]);

// Get all assets in folder (recursive)
let allInFolder = EditorEnv.assetMgr.getAllAssetsInDir(folderAsset, [AssetType.Image]);

// Get direct children
let children = EditorEnv.assetMgr.getChildrenAssets(folderAsset, [AssetType.Image]);

// Get all user assets (under assets/src/packages)
let userAssets = EditorEnv.assetMgr.getUserAssets();

// === Path Conversion ===

// Get absolute path
let fullPath = EditorEnv.assetMgr.getFullPath(asset);

// Relative path -> absolute path
let fullPath = EditorEnv.assetMgr.toFullPath("assets/textures/hero.png");

// Absolute path -> relative path
let relPath = EditorEnv.assetMgr.toRelativePath("/Users/me/project/assets/hero.png");

// Convert "editorResources/..." path to absolute path
// The second argument (allowResourcesSearch=true) is required for editorResources paths
let fullPath = EditorEnv.assetMgr.getFullPath(
    EditorEnv.assetMgr.getAsset("editorResources/my-plugin/data.json", true)
);

// === Metadata Read/Write ===
let meta = EditorEnv.assetMgr.readMeta(asset);
let meta = await EditorEnv.assetMgr.readMetaAsync(asset);
EditorEnv.assetMgr.writeMeta(asset, meta);
await EditorEnv.assetMgr.writeMetaAsync(asset, meta);
await EditorEnv.assetMgr.setMetaData(asset, { quality: 0.8 });

// === Create Assets ===
let newAsset = EditorEnv.assetMgr.createFileAsset("assets/data/config.json");
let folder = EditorEnv.assetMgr.createFolderAsset("assets/output");

// === Import Control ===
EditorEnv.assetMgr.importAsset(asset);                    // Trigger reimport
await EditorEnv.assetMgr.unpackModel(asset);               // Unpack FBX/glTF
await EditorEnv.assetMgr.waitForAssetsReady(["assets/a.png"]); // Wait for import

// === Listen for changes ===
// IMPORTANT: You MUST remove the listener in @IEditorEnv.onUnload to avoid leaks on plugin reload
function onAssetChanged(asset: IAssetInfo, flag: AssetChangedFlag) {
    console.log(`Asset ${asset.file} changed: ${flag}`);
}
EditorEnv.assetMgr.onAssetChanged.add(onAssetChanged);

// In @IEditorEnv.onUnload:
// EditorEnv.assetMgr.onAssetChanged.remove(onAssetChanged);

// === Wait for pending changes ===
await EditorEnv.assetMgr.flushChanges();
```

### Key Differences Between Processes

| Feature | Editor.assetDb (UI) | EditorEnv.assetMgr (Scene) |
|---------|---------------------|---------------------------|
| Query style | Async (Promise) | Synchronous |
| File operations | Full CRUD | Create + metadata read/write |
| Path conversion | Supports URL conversion | No URL conversion |
| Type mapping | N/A | `getAssetTypeByFileExt` / `getFileExtByAssetType` |
| Bulk access | N/A | `allAssets` dictionary |
| Icons/thumbnails | Yes | N/A |

---

## 17. Cross-Process Communication

### UI -> Scene

```typescript
// Call a static method on a scene-process class
let result = await Editor.scene.runScript("ClassName.methodName", arg1, arg2);

// Call a method on a specific node's component
let result = await Editor.scene.runNodeScript(nodeId, componentId, "methodName", args);

// Modify node properties (auto-synced)
let node = Editor.scene.getSelection()[0];
await Editor.scene.syncNode(node);
node.props.x = 100;
```

### Scene -> UI

```typescript
// Fire-and-forget message to panel
EditorEnv.postMessageToPanel("PanelName", "eventName", data);

// Request-response to panel
let result = await EditorEnv.sendMessageToPanel("PanelName", "methodName");
```

### Preview -> UI

```typescript
let EditorClient = (<any>window).EditorClient;
let result = await EditorClient.sendMessageToPanel("PanelName", "methodName");
```

### UI -> Preview

```typescript
let result = await Editor.scene.runScriptMax("ClassName.methodName", args);
```

---

## 18. I18n Support

```typescript
let myI18n = gui.Translations.create("MyPlugin", "en");
myI18n.setContent("zh-CN", {
    panelTitle: "我的插件",
    settingLabel: "启用功能"
}).setContent("en", {
    panelTitle: "My Plugin",
    settingLabel: "Enable Feature"
});

// Use in code
console.log(myI18n.t("panelTitle"));

// Use in decorators
@IEditor.panel("MyCompany.MyPlugin.MainPanel", { title: "i18n:MyPlugin:panelTitle" })
```

### Type Descriptor Captions: English Defaults, Chinese Translation

For `IEditor.FTypeDescriptor`, write English labels directly in `caption`. Only Chinese needs an additional caption translation:

```ts
export const types: IEditor.FTypeDescriptor[] = [{
    name: "MyPlugin.Config",
    caption: "Config",
    properties: [
        { name: "enabled", caption: "Enabled", type: "boolean" },
        { name: "quality", caption: "Quality", type: "number" }
    ]
}];
```

The Chinese translation map uses `type.name` as its top-level key. Within each type, `"#"` is the type caption and other keys match property names:

```ts
const zhCNTypeCaptions: Record<string, Record<string, string>> = {
    "MyPlugin.Config": {
        "#": "配置",
        enabled: "启用",
        quality: "质量"
    }
};
```

Attach the active Chinese translations in the UI process before registering the types:

```ts
@IEditor.onLoad
async onLoad() {
    // Clone shared descriptors before adding UI-only metadata.
    const editorTypes: IEditor.FTypeDescriptor[] = types.map(type => ({
        ...type,
        properties: type.properties?.map(property => ({ ...property }))
    }));

    if (i18n.language === "zh-CN") {
        for (const type of editorTypes) {
            const translation = zhCNTypeCaptions[type.name];
            if (translation)
                type.captionTranslation = translation;
        }
    }

    Editor.typeRegistry.addTypes(editorTypes);
}
```

Apply `captionTranslation` before `Editor.typeRegistry.addTypes()`. If the descriptor array is imported from a module also used by the Scene process, clone it first so UI-only translation metadata does not mutate the shared definitions.

- `caption`: English/default caption.
- `captionTranslation["#"]`: Chinese type caption.
- `captionTranslation[propertyName]`: Chinese property caption.

### Plugin Script Components: Translate Auto-Generated User Types

The previous pattern applies when the plugin owns and registers an `IEditor.FTypeDescriptor` array. A plugin script component declared with `@Laya.regClass()`, `@Laya.classInfo()`, and `@Laya.property()` is different: the editor compiler generates its user type descriptor, so the plugin must not create and register a duplicate descriptor merely to translate it.

Keep English/default text in the script decorators:

```ts
const { regClass, classInfo, property } = Laya;

@regClass()
@classInfo({ caption: "Particle Controller" })
export class ParticleController extends Laya.Script {
    @property({
        type: Number,
        caption: "Emission Rate",
        tips: "Particles emitted per second"
    })
    emissionRate = 10;
}
```

The generated user type is keyed in `Editor.typeRegistry.types` by the script asset UUID—the UUID preserved by the script's paired `.meta` when the script is renamed. Use that UUID as the top-level translation key. `"#"` translates the class/type caption; the other keys are decorated property names:

```ts
// These maps may be inline, imported, or loaded from any source.
// Their storage path and file names are not part of the API contract.
const zhCNTypeCaptions: Record<string, Record<string, string>> = {
    "01234567-89ab-cdef-0123-456789abcdef": {
        "#": "粒子控制器",
        emissionRate: "发射率"
    }
};

const zhCNTypeTips: Record<string, Record<string, string>> = {
    "01234567-89ab-cdef-0123-456789abcdef": {
        emissionRate: "每秒发射的粒子数量"
    }
};
```

User-script descriptors may already exist when the plugin UI code loads, and they are replaced when scripts are compiled or reloaded. Apply translations once immediately, then reapply them from the UI process whenever `onUserTypesChanged` fires:

```ts
class MyPluginTypeI18n {
    @IEditor.onLoad
    static onLoad() {
        if (i18n.language !== "zh-CN")
            return;

        this.applyTypeTranslations();
        Editor.typeRegistry.onUserTypesChanged.add(this.applyTypeTranslations, this);
    }

    @IEditor.onUnload
    static onUnload() {
        Editor.typeRegistry.onUserTypesChanged.remove(this.applyTypeTranslations, this);
    }

    private static applyTypeTranslations() {
        for (const typeName in zhCNTypeCaptions) {
            const type = Editor.typeRegistry.types[typeName];
            if (type)
                type.captionTranslation = zhCNTypeCaptions[typeName];
        }

        for (const typeName in zhCNTypeTips) {
            const type = Editor.typeRegistry.types[typeName];
            if (type)
                type.tipsTranslation = zhCNTypeTips[typeName];
        }
    }
}
```

The immediate call covers types registered before this loader. The event handler covers late registration and replaces translations lost when the editor rebuilds user types. Always remove the handler in `@IEditor.onUnload`. This pattern also works for plugin-owned serializable helper classes generated from Laya decorators, not only classes derived from `Laya.Script`.

---

## 19. ScriptableObject (`.sco`) Data Assets

**Version**: LayaAir 3.4.1+

**Use**: Typed, reusable serialized data that is an asset rather than a scene node or component

Use the built-in `.sco` format when a plugin needs a data resource whose fields should be edited in the normal asset Inspector and loaded as a typed `Laya.Resource`. Do not build a custom importer, exporter, saver, loader, or Inspector solely for this case; the IDE already provides that lifecycle for `Laya.ScriptableObject`.

Define the resource in a normal Laya script so the registered type is available to the Scene process and, when the asset is used by the game, to Preview/runtime code:

```ts
const { regClass, classInfo, property } = Laya;

@regClass()
@classInfo({
    caption: "Game Balance",
    menu: "My Plugin/Data",
    newAssetName: "GameBalance",
    icon: "editorResources/my-plugin/game-balance.svg"
})
export class GameBalance extends Laya.ScriptableObject {
    @property({ type: Number, caption: "Move Speed", min: 0 })
    moveSpeed = 5;

    @property({ type: String, caption: "Display Name" })
    displayName = "Default";
}
```

- `menu` is the path relative to **Project/Create**. Slash-separated segments create submenus; an empty string places the type at the Create-menu root. The normal `",order"` suffix can control ordering.
- `newAssetName` is the default file name without the `.sco` extension. If omitted, the localized type caption is used.
- `caption` is the default/English type label. For Chinese captions and property labels on this auto-generated user type, use the script-UUID translation pattern in §18 rather than registering a duplicate type descriptor.
- `icon` is optional and follows the normal `editorResources/<plugin-name>/...` path rules.

After the script compiles, **Project/Create/My Plugin/Data/Game Balance** creates `GameBalance.sco`. The initial file is JSON shaped like:

```json
{
  "_$ver": 1,
  "_$type": "<registered-script-type-id>"
}
```

Do not hand-write or rewrite `_$type`. For project scripts it identifies the generated registered type, normally through the script asset UUID, so keep the script and its `.meta` together when moving, renaming, or sharing it. The `.sco` importer uses `_$type` as the asset subtype; the Inspector then exposes the class's `@Laya.property()` fields and saves non-default values back to the same file.

Load the asset with the normal Laya loader and cast it to the registered class:

```ts
const balance = await Laya.loader.load("resources/GameBalance.sco") as GameBalance;
console.log(balance.moveSpeed);
```

Use the actual project URL or a serialized asset reference in production code. Serialized resource references inside `.sco` data are included in dependency analysis during export and build. If the class must work in game Preview/runtime, do not place its definition in a UI-only `@IEditor.*` script.

---

## 20. Package Precompilation

Configure `precompile` in the `package.json` directly inside the exported package folder. It works with **Export Installable Package** / `export-installable-package`, and also with a regular resource-package export when exactly one folder is selected. A multi-selection does not activate package precompilation.

### Shorthand and detailed options

The directory-array shorthand remains supported:

```json
{
  "name": "com.example.my-plugin",
  "version": "1.0.0",
  "precompile": ["editor", "scene"]
}
```

Use the object form when the package needs explicit compilation settings:

```json
{
  "name": "com.example.my-plugin",
  "version": "1.0.0",
  "precompile": {
    "directories": ["editor", "scene"],
    "minify": true,
    "keepNames": true,
    "define": {
      "__PLUGIN_DEBUG__": "false",
      "__PLUGIN_CHANNEL__": "\"stable\""
    },
    "external": ["lodash"]
  }
}
```

`lodash` is only an example external dependency; replace it with the packages actually used by the plugin, or omit `external` when no additional exclusions are needed.

| Property | Default | Meaning |
| --- | --- | --- |
| `directories` | Required in object form | Non-empty array of source-directory paths relative to the package root. Each must be an existing directory inside that root; absolute paths, parent traversal, and symlinks escaping the root are rejected. |
| `minify` | `true` | Boolean controlling minification of the precompiled bundles. Set `false` when readable output is needed. |
| `keepNames` | `true` | Boolean preserving function/class names during compilation, including with minification. Keep it enabled when code depends on these names; this is not source-map generation. |
| `define` | `{}` | Map of compile-time replacements. Every value must be a string containing a JavaScript expression: `"false"` inserts a boolean; `"\"stable\""` inserts a string literal. Package entries override project definitions with the same key. |
| `external` | `[]` | Array of non-empty module specifiers/patterns to leave out of the bundle, appended to project external settings and host built-ins. It does not install npm packages or declare LayaAir `pluginDependencies`. |

An omitted `precompile` or the shorthand `[]` disables precompilation. The object form requires non-empty `directories`; unknown object properties and wrong value types are rejected. Do not pass arbitrary esbuild options such as `sourcemap` or `target` into this object. Precompiled output has no source maps; its TypeScript configuration/target comes from the package's `tsconfig.json` or the IDE default.

### Output and external dependencies

- TypeScript entries are classified into UI and Scene bundles. Export writes whichever bundles contain code to `build~/bundle.editor.js` and `build~/bundle.scene.js`, and removes the listed directories and their metadata from the staged archive, not the working source folder. Existing staged `build~/` and `node_modules/` are replaced rather than copied wholesale.
- Selecting the package root (`"."`) removes all staged root content except `package.json` and its metadata before generated output is added. Prefer specific source directories when icons, locales, runtime source, or other assets must remain in the package.
- npm imports that remain external in the generated bundles are resolved from installed dependencies and copied, with their required dependency closure, into package-local `node_modules/` for IDE/CLI use. Merely listing an unused package in `external` does not include it. IDE/Node built-ins are not copied. Install ordinary external npm dependencies in the development environment before export; an unresolved external is not automatically downloaded and must be provided by its runtime host if it is not packaged.
- Dependencies containing native `.node` code require compatible platform, architecture, and Node/Electron ABI. Export warns about native code; it does not rebuild it for every target machine.

### Source boundary and Preview/runtime

Retained TypeScript outside the selected directories must not import files inside them: those source files will be absent after installation. Export checks this boundary and fails on such imports. Put shared code outside removed directories when retained source needs it, or include all its consumers in the precompiled set.

Precompilation produces UI/Scene bundles only, not Preview/runtime `bundle.js`. Keep gameplay code that must participate in Preview/game builds outside the removed directories. A runtime-classified script can enter the Scene bundle, but that does not make its source available to the game's Preview build.

---

## 21. Frame Debugger & Profiler

These are UI-process APIs mounted by their corresponding feature packs. The declaration/interface names are `IFrameDebugger` and `IProfiler`; plugin code calls the runtime values `IEditor.FrameDebugger` and `IEditor.Profiler`. They share state with the built-in panels but do not open a panel or file dialog themselves. If a deployment can omit or unload these feature packs, check that the runtime value exists before calling it.

### Frame Debugger

`IEditor.FrameDebugger.captureFrame()` captures the next WebGL frame and returns JSON-serializable Spector.js data, including ordered commands and initial/final render state. By default it captures the running scene with low-resolution screenshots; use `targetSceneView: true` to capture the editor Scene view without entering play mode, or `resolution: "none"` when screenshots are unnecessary.

```ts
const capture = await IEditor.FrameDebugger.captureFrame({
    targetSceneView: true,
    resolution: "none"
});
console.log(capture.commands);
```

Only one capture can run at a time and calls do not queue. Handle rejected errors such as `CAPTURE_BUSY`, `SCENE_NOT_PLAYING`, `NO_SCENE`, and `UNSUPPORTED_SCENE`. The exact nested capture fields depend on the bundled Spector.js version and WebGL context.

### Profiler

`IEditor.Profiler` controls a Tracy session. A basic automated workflow is `start()` → exercise the workload → `stop()` → `exportReport()` or `exportCapture()` → `dispose()`:

```ts
await IEditor.Profiler.start({ target: "editor" });
try {
    await runMeasuredWorkload();
    await IEditor.Profiler.stop();
    const report = await IEditor.Profiler.exportReport({ maxZones: 100 });
    console.log(report.zones, report.frames);
} finally {
    await IEditor.Profiler.dispose();
}
```

Targets are `"editor"`, `"browser"`, `"emulator"`, and `"remote"`. Browser profiling needs `connect({ target: "browser" })` before opening the browser preview, then `start()` after that preview registers. Remote profiling also requires `remoteAddress`. `exportCapture()` returns native `.tracy` bytes; `exportReport()` returns a bounded JSON summary of CPU zones and frame timings for automated analysis, not GPU, memory, lock, call-stack, or full timeline data. Use `state` / `onChanged` for progress and remove listeners when finished. Stop before reporting for a stable snapshot, and always call `dispose()` when the API-owned session is no longer needed.

---

## Decorator Quick Reference

| Decorator | Process | Purpose |
|-----------|---------|---------|
| `@IEditor.panel(id, options)` | UI | Register panel |
| `@IEditor.menu(path, options)` | UI | Register menu item |
| `@IEditor.regClass()` | UI | Register type for serialization |
| `@IEditor.property(type)` | UI | Declare inspectable property |
| `@IEditor.inspectorField(name)` | UI | Custom inspector field |
| `@IEditor.inspectorLayout(type, order)` | UI | Asset inspector layout |
| `@IEditor.onLoad` | UI | Plugin load lifecycle |
| `@IEditor.onUnload` | UI | Plugin unload lifecycle |
| `@IEditorEnv.regClass()` | Scene | Register scene class |
| `@IEditorEnv.customEditor(target)` | Scene | Custom gizmo editor |
| `@IEditorEnv.regBuildPlugin(platform, priority)` | Scene | Build pipeline plugin |
| `@IEditorEnv.regAssetProcessor()` | Scene | Asset import processor |
| `@IEditorEnv.regAssetImporter(exts)` | Scene | Asset importer |
| `@IEditorEnv.regAssetExporter(exts, opts)` | Scene | Asset exporter |
| `@IEditorEnv.regAssetSaver(exts)` | Scene | Asset saver |
| `@IEditorEnv.regSceneHook()` | Scene | Scene event hooks |
| `@IEditorEnv.onPreload` | Scene | Scene preload lifecycle |
| `@IEditorEnv.onLoad` | Scene | Scene load lifecycle |

## Property Decorator Options

```typescript
@IEditor.property({
    type: String,                    // String, Number, Boolean, class, or array
    caption: "Display Label",        // Display name
    tips: "Tooltip text",            // Hover tooltip
    default: "defaultValue",         // Default value
    enumSource: ["A", "B", "C"],     // Enum dropdown
    enumSource: [ { name:"A", value:"a" }, { name: "B", value: "b" }, { name: "C", value: "c" } ],  // Enum dropdown with custom values
    min: 0, max: 100, step: 1,      // Numeric constraints
    fractionDigits: 2,               // Decimal places
    isAsset: true,                   // Is asset reference
    assetTypeFilter: "Image",        // Filter asset type
    inspector: "CustomField",        // Use custom field
    hidden: "data.mode == 1",        // Conditional hide
    readonly: "data.mode == 1",      // Conditional read-only
    serializable: false,              // Exclude in serialization
})
```

## Panel Options Reference

```typescript
@IEditor.panel(id, {
    title: string,                   // Display title (supports i18n)
    icon: string,                    // Icon path
    location: "left"|"right"|"top"|"bottom"|"popup"|"embed",
    locationBase: string,            // Reference panel
    autoStart: boolean,              // Auto-open on load
    showInMenu: boolean,             // Show in Panel menu
    hotkey: string,                  // Keyboard shortcut
    transparent: boolean,            // Transparent background
    help: string,                    // Help URL
    usage: "common"|"project-settings"|"build-settings"|"preference"|"preview",
    order: number,                   // Display order, no use if usage is "common" or omitted
    stretchPriorityX: -1|0|1,       // Horizontal stretch
    stretchPriorityY: -1|0|1,       // Vertical stretch
})
```
