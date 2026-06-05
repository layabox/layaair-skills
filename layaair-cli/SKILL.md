---
name: layaair-cli
description: "Help users use the LayaAir CLI tool to create, build, or validate projects from the command line. TRIGGER when: user says 'create a layaair project', 'build a layaair project', wants CLI commands, wants to run a script/plugin function from the command line, or manages CLI versions. Do NOT trigger for editor plugin/panel development (that is layaair-ide-plugin). Chinese triggers: 创建layaair项目, 创建一个项目, 构建项目, 打包项目, 验证文件, CLI命令, layaair命令, 运行脚本, 调用插件函数, 安装layaair."
---

# LayaAir CLI Usage Skill

You are an expert at using the LayaAir CLI tool (`layaair`). Help users run the correct commands for creating projects, building for platforms, validating files, running scripts, and managing CLI versions.

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
| `build` | Build project for a target platform | `[platform]` positional, `-t/--build-platform`, `-p/--project`, `-o/--build-out`, `-r/--build-recompile`, `-l/--list-platforms` |
| `validate` | Validate resource files | `[files...]` positional, `-f/--validate-files`, `-p/--project` |
| `run` | Start built-in preview server, or run a `--script` | `-p/--project`, `--script=Class.method`, `--script-file=<file.ts>`, `--script-args`, `--disable-plugins` |

**Global options:** `-h/--help`, `-d/--debug`, `--enable-all-panels` (load all editor/extension panels in CLI mode)

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

# Validate files (positional args)
layaair validate assets/main.lh assets/player.lprefab

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
layaair help run
```
