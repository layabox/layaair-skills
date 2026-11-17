# layaair-skills

[中文](./README.zh-CN.md)

General-purpose AI Agent skills for the LayaAir engine.

[LayaAir](https://github.com/layabox/LayaAir) is a high-performance, cross-platform engine for games and interactive content. It supports 2D, 3D, UI, animation, physics, asset management, editor extensions, and multi-platform publishing. The LayaAir 3D engine provides complete 3D development capabilities, including scenes, cameras, lights, materials, meshes, skeletal animation, particles, physics, and post-processing for web, mini-game, mobile app, desktop, and other interactive projects.

This repository collects and maintains AI Agent skill instructions related to LayaAir development, helping agents understand LayaAir workflows, command-line tooling, and IDE plugin development conventions more accurately.

## Current Skills

| Skill | Description |
| --- | --- |
| `layaair-cli` | LayaAir command-line tool skill. Use it to create projects, build projects, validate resource files, start the built-in preview server, run registered script methods, and manage LayaAir CLI versions. |
| `layaair-ide-plugin` | LayaAir IDE plugin development skill. Use it to write editor plugins, editor panels, menus, inspector fields, build plugins, asset processors, scene hooks, Gizmos, and other IDE extensions. |
| [`layaair-api-knowledge`](./layaair-api-knowledge/) | Look up LayaAir APIs, compare versions, and inspect scene node and component schemas. Runs independently of the LayaAir Agent plugin. |

To install the standalone knowledge Skill, copy its entire directory into your AI client's Skill directory. The client needs Node.js 20 or newer and outbound HTTPS for the first API query, which downloads the database automatically. The per-user database cache is shared with LayaAir Agent and LayaAir MCP. Scene node and component Schema queries are available offline immediately.

## License

MIT. See [LICENSE](./LICENSE).
