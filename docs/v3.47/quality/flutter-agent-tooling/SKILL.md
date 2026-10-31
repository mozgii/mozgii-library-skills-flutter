---
name: flutter-agent-tooling
description: Connects an AI coding agent to a Flutter project - the Dart and Flutter MCP server and its tools, the official plugin for Claude Code, Cursor, Codex and Copilot, rules, package skills, and the loop of editing, hot reload and reading runtime errors. Use when setting up an agent on a Flutter project, asking what the Dart MCP server can do, fixing an agent that can't see the running app, or enabling launch_app and other disabled tools.
metadata:
  tags:
  - flutter
  - dart
  - mcp
  - agents
---
# Agent Tooling for Flutter

An agent that can only edit files is working blind. Connected to the Dart and Flutter MCP server, it can read analyzer diagnostics, see the running widget tree, hot reload its change and read the resulting exception. This skill sets that loop up and maps each tool to the job it does.

## Step 0: Check the Toolchain

```bash
flutter --version
dart --version
dart mcp-server --help
```

The server ships inside the Dart SDK and needs Dart 3.9 or later. The `dart` on the agent's `PATH` must be the one from your Flutter SDK. `rip_grep_packages` also needs ripgrep installed. The examples were compiled, not run on a device, so try hot reload and the widget tree tools against a real running app.

## Which Setup to Use

| Your agent | Do this | Why |
|---|---|---|
| Claude Code | `claude plugin marketplace add flutter/agent-plugins`, then `claude plugin install dart-flutter@dart-flutter` | The plugin brings the MCP server and skills; rules are added by hand |
| Cursor | `/add-plugin dart-flutter`, or Add to Cursor from the marketplace | The MCP server is configured for you |
| Codex | `codex plugin marketplace add flutter/agent-plugins`, then `codex plugin add dart-flutter@dart-flutter` | Same plugin, Codex format |
| GitHub Copilot in VS Code | A `.vscode/mcp.json` with a `servers` key | Copilot takes the MCP config directly |
| Windsurf, Zed, Cline and others | The `mcpServers` JSON below in the client's MCP file | Works with any MCP client |

```json
{
  "mcpServers": {
    "dart": { "command": "dart", "args": ["mcp-server"] }
  }
}
```

The plugins for Claude Code and Codex don't bundle rules. Add the ones you want to `CLAUDE.md`, `AGENTS.md` or `.agent/rules/`, and use `.cursor/rules/*.mdc` for Cursor or `.github/copilot-instructions.md` for Copilot.

## Dart MCP Server Tools

| Job | Tool | On by default |
|---|---|---|
| Find analyzer errors | `analyze_files` | yes |
| Hover, signature help, symbol search | `lsp` | yes |
| Run pub commands | `pub` | yes |
| Search pub.dev | `pub_dev_search` | yes |
| Read dependency source | `read_package_uris`, `rip_grep_packages` | yes |
| Manage project roots | `roots` | yes |
| Connect to a running app | `dtd`, `vm_service` | yes |
| Read the app's exceptions | `get_runtime_errors` | yes |
| Apply a change | `hot_reload`, `hot_restart` | yes |
| Read the widget tree | `widget_inspector` | yes |
| Tap, type, scroll | `flutter_driver_command` | yes |
| Start and stop an app | `launch_app`, `stop_app`, `list_running_apps`, `list_devices`, `get_app_logs` | no |
| Run tests, format, apply fixes | `run_tests`, `dart_format`, `dart_fix` | no |
| Create a project | `create_project` | no |
| Read the cursor position | `get_active_location` | no |

## Turning On the Disabled Tools

Pass `--enable` with a tool name or a category; enabling a category also enables the disabled tools in it. `--disable` beats `--enable` when both apply at one level, and a tool name beats its category.

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server", "--enable", "flutter_app_lifecycle", "--enable", "cli"]
    }
  }
}
```

The categories are `analysis`, `cli`, `dart_tooling_daemon`, `flutter`, `flutter_app_lifecycle`, `flutter_driver` and `package_deps`. A Dart-only program needs `--observe` to reach DTD, and `--no-dds` on a Flutter run turns DTD off along with the Dart Development Service.

## Worked Example: Fixing a Bug in the Running App

The order matters, because app tools fail until DTD is connected.

1. Start the app: call `launch_app`, or run `flutter run` yourself and copy its DTD URI (`--print-dtd` prints it).
2. Call `dtd` with command `listDtdUris`, then `connect`. `listConnectedApps` shows what's attached.
3. Reproduce the problem, then call `get_runtime_errors`.
4. Call `widget_inspector` with `get_widget_tree` to see the real tree. Selectors must come from this output, not from a guess.
5. Edit the code, then call `analyze_files` on the changed paths.
6. Call `hot_reload`, and read `get_runtime_errors` again. Use `hot_restart` when the change touched `initState`, a global or `main()`.
7. Once it's fixed, add a test so the bug stays fixed (`flutter-add-widget-test`).

## Package Skills

Packages on pub.dev can ship their own agent skills in a `skills/` folder, following the Agent Skills format. `dart run skills@ get` scans your direct dependencies and offers their skills, and `--all` takes everything. `dart run skills@ list` shows what's installed, `prune` removes skills for dropped packages, and `add <git_url>` pulls skills from a Git repository. Installed skills land in the agent's folder, such as `.agents/skills/`, and `.config/dart_skills/` tracks them, so commit both if your team shares skills. `package:skills_lint` validates a skill you publish.

The Developer Knowledge MCP server (`developers.google.com/knowledge/mcp`) is a separate service that gives an agent search over the official Flutter and Dart docs.

## Verify

```bash
claude plugin marketplace list     # Claude Code
codex plugin list                  # Codex
```

Then ask the agent to list its Dart tools and to run `analyze_files` on the project. If it reports diagnostics, the server works. Next, start the app and have the agent call `get_runtime_errors`: an empty result, with no connection error, proves the DTD link.

## Gotchas

1. **`dtdNotConnected`.** Symptom: every app tool fails. Cause: `listDtdUris` and `connect` weren't called first. Fix: call them, and the error text gives the exact `connect` call.
2. **`launch_app` is missing.** Symptom: the agent says the tool doesn't exist. Cause: it's off by default. Fix: add `--enable flutter_app_lifecycle`.
3. **`Flutter SDK cannot be found`.** Symptom: Flutter commands fail through the server. Cause: `dart` came from a standalone Dart SDK. Fix: fix `PATH`, or pass `--flutter-sdk`.
4. **The driver tool does nothing.** Symptom: `flutter_driver_command` hangs or errors. Cause: the app wasn't built with `enableFlutterDriverExtension()`. Fix: use a separate entry point (`flutter-add-integration-test`).
5. **`rip_grep_packages` fails.** Symptom: a missing binary error. Cause: ripgrep isn't installed. Fix: install it.

## Anti-Patterns

- Guessing widget selectors. Read the tree first.
- Hot restarting after every edit. Hot reload keeps state, so it's faster.
- Pasting logs into chat by hand while the server is available. Call `get_runtime_errors`.
- Editing skills installed by `skills@`. An update may overwrite them.

## Pre-Merge Checklist

- [ ] The MCP config is committed for the team's agent
- [ ] Tools the project needs are enabled, and no more
- [ ] Rules are added where the agent reads them
- [ ] `.config/dart_skills/` is committed if skills are shared

## See Also

- `flutter-add-integration-test` for turning a session into a test
- `flutter-fix-layout-issues` for reading runtime layout errors
- `dart-analyze-and-fix` for the analyzer tools
