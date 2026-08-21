---
name: layaair-cli
description: "Help users use the LayaAir CLI to create, build, validate, export .layapkg resource packages or installable packages, control package installation, run scripts, and manage CLI versions. TRIGGER when: user asks for LayaAir CLI commands, project creation/building, resource validation, package export or installation control, script/plugin execution from the command line, or CLI version management. Do NOT trigger for editor plugin/panel development (use layaair-ide-plugin). Chinese triggers: 创建layaair项目, 创建一个项目, 构建项目, 打包项目, 导出资源包, 导出可安装包, 跳过包安装, layapkg, 验证文件, CLI命令, layaair命令, 运行脚本, 调用插件函数, 安装layaair."
---

# LayaAir CLI Usage Skill

You are an expert at using the LayaAir CLI tool (`layaair`). Help users run the correct commands for creating projects, building for platforms, validating files, exporting `.layapkg` packages, running scripts, and managing CLI versions.

The information below covers all current flags and behavior. If you need to confirm flag names or discover options not listed here, run `layaair help` to get the authoritative output for the installed version.

## Installation

The install script sets up a **dispatcher only** — you must also install at least one CLI version before `layaair` is usable.

### macOS / Linux

```bash
# One-liner: install dispatcher + latest CLI runtime
curl -fsSL https://raw.githubusercontent.com/layabox/layaair-cli/master/install.sh | bash && ~/.layaair/layaair install

# Specific version
curl -fsSL https://raw.githubusercontent.com/layabox/layaair-cli/master/install.sh | bash && ~/.layaair/layaair install 3.4.0

# Custom install directory
curl -fsSL https://raw.githubusercontent.com/layabox/layaair-cli/master/install.sh | LAYAAIR_INSTALL_DIR=/opt/layaair bash && /opt/layaair/layaair install
```

**Requirements:** Node.js v20+, `unzip` command.

### Windows (PowerShell)

```powershell
# One-liner: install dispatcher + latest CLI runtime
iwr https://raw.githubusercontent.com/layabox/layaair-cli/master/install.ps1 | iex; layaair install

# Specific version
iwr https://raw.githubusercontent.com/layabox/layaair-cli/master/install.ps1 | iex; & "$env:USERPROFILE\.layaair\layaair.cmd" install 3.4.0

# Custom install directory
$env:LAYAAIR_INSTALL_DIR = "C:\tools\layaair"; iwr https://raw.githubusercontent.com/layabox/layaair-cli/master/install.ps1 | iex; & "C:\tools\layaair\layaair.cmd" install
```

**Requirements:** Node.js v20+. Default install path: `%USERPROFILE%\.layaair`.

---

## Subcommands Overview

| Subcommand | Purpose | Key flags (read `layaair help <cmd>` for full list) |
|------------|---------|----------------------------------------------|
| `create` | Create a new project from a template | `[name]` positional, `-n/--create-name`, `-p/--create-path`, `-s/--create-subdir`, `-t/--create-template`, `-l/--list-templates` |
| `build` | Build project for a target platform | `[platform]` positional, `-t/--build-platform`, `-p/--project`, `-o/--build-out`, `-r/--build-recompile`, `-l/--list-platforms`, `--skip-package-install` |
| `validate` | Validate resource files | `[files...]` positional, `-f/--validate-files`, `-p/--project`, `--skip-package-install` |
| `export-package` | Export one or more assets as a regular `.layapkg` resource package | `<project-relative-asset...>` positional, `-o/--output=<project-relative-file>`, `-p/--project`, `--include-dependencies`, `--disable-plugins=<bool>`, `--skip-package-install` |
| `export-installable-package` | Export one package folder as an installable `.layapkg` | `<project-relative-folder>` positional, `-o/--output=<project-relative-file>`, `-p/--project`, `--disable-plugins=<bool>`, `--skip-package-install` |
| `run` | Start built-in preview server, or run a `--script` | `-p/--project`, `--script=Class.method`, `--script-file=<file.ts>`, `--script-args`, `--disable-plugins`, `--skip-package-install` |

**Common options:** `-h/--help`, `-d/--debug`, `--enable-all-panels` (load all editor/extension panels in CLI mode), `--skip-package-install` (for commands that open a project; not `create`)

The `run` subcommand can also be invoked without spelling it out: `layaair [options]` is equivalent to `layaair run [options]`.

---

## Behavioral Notes

### `create` — List templates before picking one

When the user asks for a specific template type, **do NOT read files from `~/.layaair/` or the installation directory** to find template names. Always get the live template list from the CLI:

```bash
layaair create -l
# or
layaair create --list-templates
```

This prints all available templates (builtin, cloud, local). Use the exact English display name as the `--create-template` value. If the user's intent is ambiguous, show them the list and ask.

**Network access required for full list:** `--list-templates` fetches the cloud template catalog at startup. Without network access, only builtin templates and previously-downloaded cloud templates (local cache) are shown — cloud-only entries that haven't been downloaded yet will be missing. If running in a sandboxed environment, grant network access before listing or creating from a cloud template.

**Workflow when user specifies a template type:**
1. Run `layaair create -l` to get the live list
2. Match the user's intent to a template display name
3. Run `layaair create <name> -t "<exact display name>"` (or `layaair create -n <name> -t "<exact display name>"`)

### `create` — No post-create steps needed
After `layaair create` succeeds, the project is ready. **Do NOT automatically run `layaair build`, start the preview server, or any other command as a follow-up.** Each of these is an independent workflow that the user will invoke explicitly when they need it — do not chain them onto a create unless the user specifically asked for it.

### `create` — Default is direct in the target directory
`--create-subdir` defaults to `false`, meaning project files go directly into the target directory. **Do NOT add `--create-subdir` unless the user explicitly asks for a subdirectory.** Most users want files in the current/target directory directly.

### `build` — platform is positional; use `-l` to discover options

Platform can be passed as a positional argument or via `-t/--build-platform`. To see valid platform names for a project:

```bash
layaair build -l
# or
layaair build --project=<path> --list-platforms
```

### Project commands — control package installation

By default, commands that open a project may reconcile its package manifest, install or update packages, and apply pending package transactions. Add `--skip-package-install` to `build` (including `--list-platforms`), `validate`, `run`, `export-package`, or `export-installable-package` when the command must use only packages already installed locally:

```bash
layaair build web -p . --skip-package-install
layaair run -p . --skip-package-install
```

With this flag, the CLI reads the existing `package-lock.json` and recognizes only packages whose cached directories are present. It does not install, update, download, or apply pending package transactions. Therefore, use it only when all required packages are already installed; otherwise package-provided assets or functionality may be unavailable.

This flag is independent of plugin loading: installed package plugins can still load. To skip both package installation/update and user/package plugins, combine it with `--disable-plugins` on commands that support that option.

When installation or update needs the LayaAir store, CLI mode does not use an interactive editor login. Supply the API key through `LAYAAIR_API_KEY`; if no store access is intended, use `--skip-package-install` instead:

```bash
# macOS / Linux
LAYAAIR_API_KEY=<api-key> layaair build web -p .

# Windows PowerShell
$env:LAYAAIR_API_KEY = "<api-key>"; layaair build web -p .
```

### Package export — choose the correct package type

Use `export-package` for an ordinary resource package. It accepts one or more files or folders and can optionally include referenced assets:

```bash
layaair export-package assets/MyPackage -o output/MyPackage.layapkg
layaair export-package assets/UI assets/Hero.lh -o output/game-assets.layapkg -p /tmp/demo --include-dependencies
```

Use `export-installable-package` when the result is meant to be installed as a package. It accepts exactly one folder inside the project, that folder must directly contain a valid `package.json`, and the folder's contents are placed at the archive root. The folder can be outside `assets` and does not need to be an imported asset:

```bash
layaair export-installable-package packages/MyPackage -o output/MyPackage.layapkg
```

Do not substitute a generic `package` subcommand; the command names are exactly `export-package` and `export-installable-package`.

### Package export — paths, output, and plugin loading

- `-o/--output` is required. A relative output path is resolved from the project root selected by `--project`, not from the shell's current working directory. An absolute output path is also accepted.
- `-p/--project` defaults to the current working directory. Relative project paths are also resolved from the current working directory.
- For `export-package`, relative asset arguments start at the project root and must include the `assets/` prefix, for example `assets/UI/Hero.lh`. Bare paths relative to the assets directory, such as `UI/Hero.lh`, are not accepted. An absolute asset path is accepted only when it is inside the selected project's `assets` directory.
- For `export-installable-package`, the single folder argument is relative to the project root and must stay inside that project. It is not restricted to `assets`; for example, use `packages/MyPackage`. An absolute folder path is accepted only when it is inside the project.
- Package export skips user and package plugins by default. Pass `--disable-plugins=false` only when the export needs those plugins. This differs from `run`, where plugins are loaded unless `--disable-plugins` is supplied.
- `--include-dependencies` applies only to `export-package`; it adds referenced project assets while excluding built-in, internal, package, and memory assets.

### Package export — optional precompilation

Precompilation is considered only when the export selects exactly one folder and that folder directly contains `package.json`. If its `precompile` property is a non-empty array of directories, those directories are compiled and replaced in the staged package by generated bundles under `build~`.

For `export-package`, `package.json` is optional and only controls this precompilation behavior. For `export-installable-package`, a valid direct-child `package.json` is required even when no `precompile` property is present.

### `run` — Built-in preview server or script runner

`layaair run` (or just `layaair`) **without `--script`** starts the built-in HTTP/HTTPS preview server. No external web server is needed.

```bash
layaair run -p /path/to/myproject
# equivalent shorthand:
layaair -p /path/to/myproject
```

The server port is read from the project's EditorSettings. Once running, open the printed URL in a browser to preview the project.

**Note:** The preview server binds to a TCP port. If running in a sandboxed environment, make sure the sandbox allows outbound/inbound port binding before starting the server.

### `run --script` — Run any registered class method
The `--script=ClassName.methodName` flag executes a static method on a class registered in the project. It works for **user/plugin code** — any class registered with `@IEditorEnv.regClass()` is callable:

```typescript
@IEditorEnv.regClass()
export class MyCLITools {
    static async exportData(outputPath: string): Promise<void> {
        // ... your logic
    }
}
```

```bash
layaair run -p . --script=MyCLITools.exportData --script-args="/tmp/out.json"
```

`--script-args` is a single quoted string; the CLI splits it on spaces (quote-aware) and passes each token as a positional argument.

### `run --script-file` — Compile an extra TypeScript file for this run only

`--script-file=<file.ts>` compiles an additional `.ts` or `.tsx` file for this CLI run. The file is **not** imported into the project's asset database. It can be used with or without `--script`.

```bash
# With --script: define the class in an external file and call it
layaair run -p . --script=AX.test --script-file=/tmp/a.ts

# Without --script: just compile/load extra code during preview server startup
layaair run -p /tmp/demo --script-file=/tmp/MyClass.ts
```

- Path can be absolute or relative to the **current working directory** (not the project root).
- Accepts comma-separated values for multiple files: `--script-file=a.ts,b.ts`
- Must be a `.ts` or `.tsx` file; must exist on disk. The CLI throws an error otherwise.

### `run --disable-plugins` — Skip user and package plugins

`--disable-plugins` prevents user plugins and package plugins from loading during this run. Useful for isolating issues or running in a clean environment.

```bash
layaair run -p /tmp/demo --disable-plugins
```

---

## Version Management

```bash
layaair install [version]        # Omit version for latest
layaair uninstall <version>      # Remove a version
layaair list                     # List installed (* = newest)
layaair --version                # Print active version
layaair --version=3.4.0 build .. # Use a specific version for this run
```

The dispatcher auto-selects the newest installed version. It also reads the project's `.laya` file to infer a version when `--project` is set.

---

## Quick Reference

```bash
# Install (macOS/Linux)
curl -fsSL https://raw.githubusercontent.com/layabox/layaair-cli/master/install.sh | bash && ~/.layaair/layaair install
# Install (Windows PowerShell)
# iwr https://raw.githubusercontent.com/layabox/layaair-cli/master/install.ps1 | iex; layaair install

# List available templates (do this before --create-template)
layaair create -l

# Create a project (default template, positional name)
layaair create MyGame

# Create with a specific template
layaair create MyGame -t "2D empty project"

# List valid build platforms for a project
layaair build -l

# Build for web (positional platform)
layaair build web

# Build for web specifying project path
layaair build web -p /tmp/demo

# Build using only packages already installed locally
layaair build web -p . --skip-package-install

# Validate files (positional args)
layaair validate assets/main.lh assets/player.lprefab

# Export one or more assets as a regular resource package
layaair export-package assets/MyPackage -o output/MyPackage.layapkg

# Export a regular package and include referenced assets
layaair export-package assets/UI assets/Hero.lh -o output/game-assets.layapkg -p /tmp/demo --include-dependencies

# Export exactly one folder as an installable package
# packages/MyPackage must directly contain package.json
layaair export-installable-package packages/MyPackage -o output/MyPackage.layapkg

# Start built-in preview server (no external web server needed)
# Note: requires sandbox to allow port binding
layaair run -p .

# Run a custom script function
layaair run -p . --script=MyExporter.run --script-args="output.zip"

# Run a script with an extra TypeScript file (not in the project asset database)
layaair run -p . --script=AX.test --script-file=/tmp/a.ts

# Check available flags for a subcommand
layaair help create
layaair help build
layaair help validate
layaair help export-package
layaair help export-installable-package
layaair help run
```
