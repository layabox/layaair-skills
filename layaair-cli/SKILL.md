---
name: layaair-cli
description: "Help users use the LayaAir CLI tool to create, build, or validate projects from the command line. TRIGGER when: user says 'create a layaair project', 'build a layaair project', wants CLI commands, wants to run a script/plugin function from the command line, or manages CLI versions. Do NOT trigger for editor plugin/panel development (that is layaair-ide-plugin). Chinese triggers: 创建layaair项目, 创建一个项目, 构建项目, 打包项目, 验证文件, CLI命令, layaair命令, 运行脚本, 调用插件函数, 安装layaair."
---

# LayaAir CLI Usage Skill

You are an expert at using the LayaAir CLI tool (`layaair`). Help users run the correct commands for creating projects, building for platforms, validating files, running scripts, and managing CLI versions.

The information below covers all current flags and behavior. If you need to confirm flag names or discover options not listed here, run `layaair help` to get the authoritative output for the installed version.

## Installation

The install script sets up a **dispatcher only** — you must also install at least one CLI version before `layaair` is usable.

```bash
# Step 1: install the dispatcher + shim (goes to ~/.layaair/)
curl -fsSL https://ldc-1251285021.file.myqcloud.com/layaair3/install.sh | bash

# Step 2: install a CLI version (open a new terminal first, or prefix with ~/.layaair/)
layaair install          # latest
layaair install 3.4.0   # specific version

# One-liner
curl -fsSL https://ldc-1251285021.file.myqcloud.com/layaair3/install.sh | bash && ~/.layaair/layaair install 3.4.0
```

**Requirements:** Node.js v20+, `unzip` command.

---

## Subcommands Overview

| Subcommand | Purpose | Key flags (read `layaair help` for full list) |
|------------|---------|----------------------------------------------|
| `create` | Create a new project from a template | `--create-name` (required), `--create-path`, `--create-subdir`, `--create-template`, `--list-templates` |
| `build` | Build project for a target platform | `--build-platform` (required), `--build-out`, `--build-recompile`, `--list-platforms` |
| `validate` | Validate resource files | `--validate-files` (comma-separated, required) |
| (no subcommand) | Start built-in preview server, or run a `--script` | `--project`, `--script=Class.method`, `--script-args`, `--debug` |

---

## Behavioral Notes

### `create` — List templates before picking one

When the user asks for a specific template type, **do NOT read files from `~/.layaair/` or the installation directory** to find template names. Always get the live template list from the CLI:

```bash
layaair create --list-templates
```

This prints all available templates (builtin, cloud, local). Use the exact English display name as the `--create-template` value. If the user's intent is ambiguous, show them the list and ask.

**Network access required for full list:** `--list-templates` fetches the cloud template catalog at startup. Without network access, only builtin templates and previously-downloaded cloud templates (local cache) are shown — cloud-only entries that haven't been downloaded yet will be missing. If running in a sandboxed environment, grant network access before listing or creating from a cloud template.

**Workflow when user specifies a template type:**
1. Run `layaair create --list-templates` to get the live list
2. Match the user's intent to a template display name
3. Run `layaair create --create-name=<name> --create-template="<exact display name>"`

### `create` — No post-create steps needed
After `layaair create` succeeds, the project is ready. **Do NOT automatically run `layaair build`, start the preview server, or any other command as a follow-up.** Each of these is an independent workflow that the user will invoke explicitly when they need it — do not chain them onto a create unless the user specifically asked for it.

### `create` — Default is direct in the target directory
`--create-subdir` defaults to `false`, meaning project files go directly into the target directory. **Do NOT add `--create-subdir` unless the user explicitly asks for a subdirectory.** Most users want files in the current/target directory directly.

### `build` — `--build-platform` is required; use `--list-platforms` to discover options
`--build-platform` is now required. Omitting it causes an error. To see valid platform names for a project:

```bash
layaair build --project=<path> --list-platforms
```

### No subcommand — Built-in preview server

Running `layaair` (or `layaair --project=<path>`) **without any subcommand and without `--script`** starts the built-in HTTP/HTTPS preview server. No external web server is needed.

```bash
layaair --project=/path/to/myproject
```

The server port is read from the project's EditorSettings. Once running, open the printed URL in a browser to preview the project.

**Note:** The preview server binds to a TCP port. If running in a sandboxed environment, make sure the sandbox allows outbound/inbound port binding before starting the server.

### `--script` — Run any registered class method
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
layaair --project=. --script=MyCLITools.exportData --script-args="/tmp/out.json"
```

`--script-args` is a single quoted string; the CLI splits it on spaces (quote-aware) and passes each token as a positional argument.

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
# Install
curl -fsSL https://ldc-1251285021.file.myqcloud.com/layaair3/install.sh | bash && ~/.layaair/layaair install

# List available templates (do this before --create-template)
layaair create --list-templates

# Create a project (default template)
layaair create --create-name=MyGame

# Create with a specific template
layaair create --create-name=MyGame --create-template="2D empty project"

# List valid build platforms for a project
layaair build --project=. --list-platforms

# Build for web
layaair build --build-platform=web

# Validate files
layaair validate --validate-files=main.ls,ui.lh

# Start built-in preview server (no external web server needed)
# Note: requires sandbox to allow port binding
layaair --project=.

# Run a custom script function
layaair --project=. --script=MyExporter.run --script-args="output.zip"

# Check available flags for any subcommand
layaair help
```
