# LayaAir IDE Plugin API Patterns Reference

Complete code examples for all plugin types. Read the relevant section based on what the user needs.

> **Important: Type Name Uniqueness**
> 
> `Editor.typeRegistry.addTypes` 和 `InspectorPanel.inspect` 中的类型名（`name` 字段）注册到全局类型表，必须唯一。命名时加上插件专属前缀，如 `"MyPlugin_ConfigType"` 而非 `"ConfigType"`。
> 
> 下面示例中的类型名仅作演示，实际开发时请替换为带前缀的名称。

## Table of Contents
1. [Panel Plugin](#1-panel-plugin)
2. [React Panel](#2-react-panel)
3. [Menu Plugin](#3-menu-plugin)
4. [Dialog](#4-dialog)
5. [Inspector Field](#5-custom-inspector-field)
6. [Inspector Layout (Asset Config)](#6-inspector-layout)
7. [Settings & Preferences](#7-settings--preferences)
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
19. [Programmatic UI](#19-programmatic-ui-creation)

---

## 1. Panel Plugin

**Process**: UI  
**Use**: Custom editor panel with FairyGUI widget

```typescript
@IEditor.panel("MyPanel", {
    title: "My Panel",
    icon: "editorResources/my-plugin/icon.svg",
    location: "right",        // "left"|"right"|"top"|"bottom"|"popup"|"embed"
    locationBase: "ScenePanel", // Reference panel for positioning
    autoStart: false,          // Auto-open on project load
    showInMenu: true,          // Show in Panel menu
})
export class MyPanel extends IEditor.EditorPanel {
    async create() {
        this._panel = await gui.UIPackage.createWidget(
            "editorResources/my-plugin/MyPanel.widget"
        );
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
        // Cleanup
    }
}
```

### Panel with InspectorPanel (config-driven UI)

```typescript
@IEditor.panel("ConfigPanel", { title: "Config Panel" })
export class ConfigPanel extends IEditor.EditorPanel {
    declare _panel: IEditor.InspectorPanel;
    private _data: any;

    async create() {
        this._panel = IEditor.GUIUtils.createInspectorPanel();

        Editor.typeRegistry.addTypes([{
            name: "ConfigPanelType",
            properties: [
                { name: "text", type: "string" },
                { name: "count", type: "number", min: 0, max: 100 },
                { name: "enabled", type: "boolean", default: true },
                { name: "color", type: "color" },
                { name: "asset", type: "string", isAsset: true, assetTypeFilter: "Image" }
            ]
        }]);

        this._data = IEditor.DataWatcher.watch({});
        this._panel.inspect(this._data, "ConfigPanelType");
    }
}
```

### Panel with regClass type definition

```typescript
@IEditor.regClass()
export class MyPanelType {
    @property(String)
    text: string;

    @property(Number)
    count: number;
}

@IEditor.panel("TypedPanel", { title: "Typed Panel" })
export class TypedPanel extends IEditor.EditorPanel {
    declare _panel: IEditor.InspectorPanel;
    private _comp: IEditor.DataComponent;

    async create() {
        this._panel = IEditor.GUIUtils.createInspectorPanel();
        this._comp = new IEditor.DataComponent(MyPanelType);
        this._panel.inspect(this._comp.props, MyPanelType);
    }
}
```

---

## 2. React Panel

**Process**: UI  
**Use**: Modern React-based editor panel

> **React is built-in to the IDE.** Just `import { useState } from "react"` and use JSX directly — no `npm install react` needed. The only prerequisite is ensuring `"jsx": "react-jsx"` is set in the project's `tsconfig.json`.

### CSS Workflow

The IDE build pipeline has a **built-in css-text esbuild plugin** that imports `.css` files as strings. No extra setup needed.

**How it works:**
1. `import styles from './MyPlugin.css'` → returns the CSS content as a `string`
2. Pass to `reactDOM.adoptStyles(styles)` → injected into Shadow DOM via `adoptedStyleSheets`

**Best practices (priority order — prefer built-in first):**

> **Rule: Always try the built-in theme before writing custom CSS.**
> The base stylesheet already styles all standard HTML elements and provides component classes + layout utilities that match the editor's look. Most panels need **zero** custom CSS. Only create a custom CSS file when the built-in classes are genuinely insufficient.

| Priority | Approach | When to use | Example |
|---|---|---|---|
| 1st | **Built-in theme only** | Buttons, inputs, lists, tabs, panels, layout — covers most plugins | Use `<button className="primary">`, `.list-item`, `.flex.gap-2`, etc. No CSS file needed, no `adoptStyles()` call |
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

> **Important:** For most plugins, the built-in theme classes (`.flex`, `.gap-2`, `.panel`, `button.primary` etc.) are **sufficient and preferred**. They ensure visual consistency with the editor. Only create custom CSS when the built-in classes genuinely cannot cover your needs.

> **Tip:** When writing custom CSS, always reference the built-in CSS variables (`var(--bg-base)`, `var(--text)`, `var(--border)`, etc.) to keep your UI consistent with the editor theme.

### Basic React Panel

```typescript
import styles from "./MyPlugin.css";

function MyApp() {
    let [data, setData] = useState("");
    return (
        <div className="container">
            <input value={data} onChange={e => setData(e.target.value)} />
        </div>
    );
}

@IEditor.panel("MyReactPanel", {
    title: "React Panel",
    location: "right"
})
export class MyReactPanel extends IEditor.EditorPanel {
    private _react: IEditor.IReactDOM;

    async create() {
        this._panel = new gui.Widget();
        this._panel.setSize(600, 500);
        this._react = new IEditor.ReactDOM();
        this._react.adoptStyles(styles);           // Inject CSS
        this._react.makeFullSize(this._panel, true); // Fill parent
        this._panel.addChild(this._react);
        this._react.render(<MyApp />);
    }

    onDestroy() {
        this._react?.dispose();
    }
}
```

### React with Editor Events (External Store)

```typescript
const selectionStore = IEditor.ReactDOM.createStore<any[]>([]);

function SelectionView() {
    let items = useSyncExternalStore(selectionStore.subscribe, selectionStore.get);
    return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

@IEditor.panel("SelectPanel", { title: "Selection" })
export class SelectPanel extends IEditor.EditorPanel {
    private _react: IEditor.IReactDOM;

    async create() {
        this._panel = new gui.Widget();
        this._react = new IEditor.ReactDOM();
        this._react.makeFullSize(this._panel, true);
        this._panel.addChild(this._react);
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

Two approaches depending on UI mode. Editor-only images should always go in `editorResources/` to exclude from game build.

Supported formats: `.png`, `.jpg`, `.gif`, `.svg`, `.webp`, `.ico`, `.bmp`.

**React panels** — `import` returns an absolute `file://` URL string:
```tsx
import icon from './icon.png';
import logo from './logo.svg';

function Header() {
    return (
        <div className="flex items-center gap-2">
            <img src={icon} width={16} height={16} />
            <img src={logo} />
        </div>
    );
}
```

CSS `url()` relative paths are also auto-resolved:
```css
.icon-btn {
    background-image: url(./icon.png);
    background-size: 16px 16px;
}
```

**Built-in UI (FairyGUI)** — use `editorResources/` paths directly:
```typescript
// Panel icon in decorator
@IEditor.panel("MyPanel", {
    icon: "editorResources/my-plugin/icon.svg",
    // ...
})

// Load image in code
let loader = new gui.GLoader();
loader.url = "editorResources/my-plugin/icon.png";
```

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

### Embedding FairyGUI Inspector in React

```typescript
function InspectorEmbed() {
    let panel = IEditor.GUIUtils.createInspectorPanel();
    let data = DataWatcher.watch({ name: "Data", speed: 10 });
    panel.inspect(data, {
        name: "DemoType",
        properties: [
            { name: "name", type: String },
            { name: "speed", type: Number, min: 0, max: 100 }
        ]
    });
    panel.allowUndo = true;
    panel.resizeToFit();

    let ref = IEditor.ReactDOM.useWidget(panel);
    return <div ref={ref} style={{ height: panel.height }} />;
}
```

### Built-in Theme Reference

ReactDOM automatically injects a dark-theme base stylesheet into the Shadow DOM. Plugins can use these CSS variables, styled elements, and class names out of the box.

#### CSS Custom Properties

Override any variable in your own CSS via `:host { --variable: value; }`.

| Variable | Usage |
|---|---|
| `--bg-darkest` | Deepest background layer |
| `--bg-dark` | Dark background |
| `--bg-base` | Standard panel background |
| `--bg-elevated` | Elevated surfaces (tooltips, popups) |
| `--bg-surface` | Buttons, cards |
| `--bg-hover` | Button/control hover background (NOT for list items) |
| `--bg-pressed` | Pressed / active state |
| `--bg-input` | Input field background |
| `--accent` | Accent color (focus rings, checkboxes, links, selections) |
| `--accent-hover` | Accent hover state |
| `--accent-muted` | List item / row hover background |
| `--accent-strong` | Strong highlight (selection) |
| `--primary` | Primary button background |
| `--primary-hover` | Primary button hover |
| `--primary-text` | Primary button text |
| `--text` | Default text color |
| `--text-bright` | Bright text (headings, active tab) |
| `--text-muted` | Secondary text (placeholders, inactive hints) |
| `--text-disabled` | Disabled text |
| `--border` | Default border color |
| `--border-light` | Light border |
| `--border-hover` | Border on hover |
| `--radius` | Default border radius |
| `--radius-lg` | Large radius (inputs, primary button) |
| `--radius-sm` | Small radius (icon buttons) |
| `--font-size` | Base font size |
| `--font-family` | Font stack |
| `--scrollbar-size` | Scrollbar width/height |
| `--scrollbar-thumb` | Scrollbar thumb color |

#### Auto-styled HTML Elements

These elements are styled automatically — just use the raw HTML tag:

- `<button>` — dark surface, hover/active/disabled states
- `<input type="text|number|search|password|url|email">` — dark input style, focus ring
- `<textarea>` — multi-line input, resizable
- `<input type="checkbox">` — custom styled checkbox, accent color when checked
- `<select>` — custom dropdown arrow
- `<table>`, `<th>`, `<td>`, `<tr>` — styled table with hover rows
- `<a>` — accent colored link
- `<code>`, `<pre>` — monospace with dark background
- `<h1>`–`<h6>` — bright text, descending sizes
- `<hr>` — subtle divider

#### Component Classes

| Class | Description |
|---|---|
| `button.primary` / `.btn-primary` | Primary button (`--primary` blue, pressed: dark) |
| `button.icon` / `.btn-icon` | Small square icon button, no border, transparent bg |
| `.list-item` | List row with hover/selected states. Add `.selected` or `aria-selected="true"` |
| `.tab` | Tab button. Add `.active` or `aria-selected="true"` for current tab |
| `.panel` | Bordered container section |
| `.panel-header` | Panel title bar (bold text, bottom border) |
| `.panel-body` | Panel content area (padded) |
| `.menu` / `[role="menu"]` | Popup menu container |
| `.menu-item` / `[role="menuitem"]` | Menu row. Add `.disabled` or `aria-disabled="true"` |
| `.menu-divider` / `[role="separator"]` | Menu separator line |
| `.tooltip` / `[role="tooltip"]` | Tooltip popup |
| `.badge` | Small rounded pill label (accent bg) |
| `.label` | Form label (default text color, no-select) |

#### Utility Classes

| Class | Effect |
|---|---|
| `.flex` | `display: flex` |
| `.flex-col` | `flex-direction: column` |
| `.flex-row` | `flex-direction: row` |
| `.flex-1` | `flex: 1; min-width: 0; min-height: 0` |
| `.flex-wrap` | `flex-wrap: wrap` |
| `.items-center` | `align-items: center` |
| `.justify-between` | `justify-content: space-between` |
| `.justify-center` | `justify-content: center` |
| `.gap-1` / `.gap-2` / `.gap-3` | Gap: small / medium / large |
| `.p-1` / `.p-2` / `.p-3` | Padding: small / medium / large |
| `.px-1` / `.px-2` | Horizontal padding: small / medium |
| `.py-1` / `.py-2` | Vertical padding: small / medium |
| `.m-1` / `.m-2` | Margin: small / medium |
| `.w-full` / `.h-full` | Width/height 100% |
| `.overflow-auto` | `overflow: auto` |
| `.overflow-hidden` | `overflow: hidden` |
| `.truncate` | Ellipsis text overflow |
| `.text-center` / `.text-right` | Text alignment |
| `.text-muted` / `.text-bright` | Muted / bright text color |
| `.text-sm` / `.text-lg` | Smaller / larger font size |
| `.hidden` | `display: none` |
| `.pointer` | `cursor: pointer` |
| `.select-none` | `user-select: none` |

#### Example: Using Built-in Styles

```tsx
function SettingsPanel() {
    let [tab, setTab] = useState("general");
    return (
        <div className="flex-col h-full">
            {/* Tabs */}
            <div className="flex">
                <button className={`tab ${tab === "general" ? "active" : ""}`}
                    onClick={() => setTab("general")}>General</button>
                <button className={`tab ${tab === "advanced" ? "active" : ""}`}
                    onClick={() => setTab("advanced")}>Advanced</button>
            </div>

            {/* Content */}
            <div className="flex-1 overflow-auto p-2 flex-col gap-2">
                <div className="panel">
                    <div className="panel-header">Settings</div>
                    <div className="panel-body flex-col gap-2">
                        <div className="flex items-center gap-2">
                            <span className="label">Name</span>
                            <input type="text" placeholder="Enter name..." />
                        </div>
                        <div className="flex items-center gap-2">
                            <input type="checkbox" id="enabled" />
                            <label htmlFor="enabled">Enable feature</label>
                        </div>
                        <div className="flex gap-1 justify-between">
                            <button>Cancel</button>
                            <button className="primary">Save</button>
                        </div>
                    </div>
                </div>
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

```typescript
let menu = IEditor.Menu.create([
    { label: "Action 1", click: () => console.log("action1") },
    { label: "Action 2", click: () => console.log("action2") },
    { type: "separator" },
    { label: "Action 3", click: () => console.log("action3") }
]);
menu.show();
```

---

## 4. Dialog

**Process**: UI

```typescript
export class MyDialog extends IEditor.Dialog {
    async create() {
        this.contentPane = await gui.UIPackage.createWidget(
            "editorResources/my-plugin/MyDialog.widget"
        );
        this.title = "My Dialog";
        this.resizable = true;
    }

    onShown() { /* dialog visible */ }
    onHide() { /* dialog hidden */ }
}

// Show dialog
Editor.showDialog(MyDialog, null);
```

### Dialog with InspectorPanel

```typescript
export class ConfigDialog extends IEditor.Dialog {
    private _data: any;

    async create() {
        let panel = IEditor.GUIUtils.createInspectorPanel();
        panel.allowUndo = true;
        panel.setSize(500, 400);
        this.contentPane = panel;
        this.title = "Configuration";
        this.resizable = true;

        this._data = IEditor.DataWatcher.watch({ name: "default", value: 100 });
        panel.inspect(this._data, {
            name: "ConfigType",
            properties: [
                { name: "name", type: String },
                { name: "value", type: Number }
            ]
        });
    }
}
```

---

## 5. Custom Inspector Field

**Process**: UI  
**Use**: Custom property editor in Inspector panel

```typescript
@IEditor.inspectorField("MyCustomField")
export class MyCustomField extends IEditor.PropertyField {
    // Optional: preload resources
    @IEditor.onLoad
    static async onLoad() {
        await gui.UIPackage.resourceMgr.load("editorResources/MyField.widget");
    }

    create(): IEditor.IPropertyFieldCreateResult {
        let input = IEditor.GUIUtils.createTextInput();
        input.on("submit", () => {
            this.target.setValue(input.text);
        });
        return {
            ui: input,
            stretchWidth: true,
            captionDisplay: "normal",  // "normal"|"hidden"|"none"
        };
    }

    refresh(): Promise<void> {
        let value = this.target.getValue();
        // Update UI to reflect current value
        return Promise.resolve();
    }
}

// Usage in a component:
// @property({ type: Laya.Node, inspector: "MyCustomField" })
// public myNode: Laya.Node;
```

### Button-style Inspector Field

```typescript
@IEditor.inspectorField("ActionButton")
export class ActionButton extends IEditor.ButtonsField {
    create() {
        let res = super.create();

        this.addButton("Execute").onClick(async () => {
            let owner = this.inspector.target.owner;
            let compId = this.inspector.target.id;
            await Editor.scene.runScript("MyHelper.execute", owner, compId);
        });

        this.addButton("Clear").onClick(async () => {
            await Editor.scene.runScript("MyHelper.clear",
                this.inspector.target.owner, this.inspector.target.id);
        });

        return res;
    }
}
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

```typescript
@IEditor.panel("MyPluginPrefs", {
    usage: "preference",       // "preference"|"project-settings"|"build-settings"
    title: "My Plugin"
})
export class MyPluginPrefs extends IEditor.EditorPanel {
    async create() {
        let panel = IEditor.GUIUtils.createInspectorPanel();
        panel.inspect(
            Editor.getSettings("MyPluginSettings").data,
            "MyPluginSettings"
        );
        this._panel = panel;
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

```typescript
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
@IEditor.panel("MyPlatformBuildSettings", {
    usage: "build-settings",
    title: "My Platform"
})
export class MyPlatformBuildSettings extends IEditor.EditorPanel {
    async create() {
        let panel = IEditor.GUIUtils.createInspectorPanel();
        panel.inspect(
            Editor.getSettings("MyPlatformSettings").data,
            "MyPlatformSettings"
        );
        this._panel = panel;
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

```typescript
// UI process
@IEditor.panel("ABCPreview", { usage: "preview" })
export class ABCPreview extends IEditor.EditorPanel implements IEditor.IPreviewPanel {
    async create() {
        this._panel = new gui.Widget();
    }

    accept(asset: IEditor.IAssetInfo): boolean {
        return asset.ext === "abc";
    }

    async refresh(asset: IEditor.IAssetInfo, render3DCanvas: IEditor.IRender3DCanvas) {
        return render3DCanvas.createObject("ABCPreviewScript", "setAssetById", asset.id);
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

// Convert "editorResources/..." path to absolute path
// The second argument (allowResourcesSearch=true) is required for editorResources paths
let fullPath = Editor.assetDb.getFullPath(
    await Editor.assetDb.getAsset("editorResources/my-plugin/locales", true)
);

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
@IEditor.panel("MyPanel", { title: "i18n:MyPlugin:panelTitle" })
```

---

## 19. Programmatic UI Creation

### GUIUtils factory methods

```typescript
let button = IEditor.GUIUtils.createButton();
let checkbox = IEditor.GUIUtils.createCheckbox();
let textInput = IEditor.GUIUtils.createTextInput();
let comboBox = IEditor.GUIUtils.createComboBox();
let colorInput = IEditor.GUIUtils.createColorInput();
let resourceInput = IEditor.GUIUtils.createResourceInput();
let inspectorPanel = IEditor.GUIUtils.createInspectorPanel();
```

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
