---
title: "Hot Reload After Dart Edits"
author: MOZGII, Inc.
lang: en
kind: rule
description: "When an agent edits Dart under lib/ while an app is running: when to hot reload, when to hot restart, and when to do nothing."
tags: ["flutter", "agents", "hot-reload", "mcp"]
---
# Hot Reload After Dart Edits

An agent working beside a running app keeps the screen in step with the code. After a Dart edit under `lib/`, apply it to the running app through the Dart and Flutter MCP server, so a broken change shows up at once.

## When to Skip Reloading

Do nothing for a change outside `lib/`, such as `test/`, `integration_test/`, `test_driver/`, `benchmark/` or `example/`. Do nothing when the edit touches only comments, doc comments or whitespace.

## Finding the App

Call the `dtd` tool with `listDtdUris`, then `connect`. `list_running_apps` covers apps started through `launch_app`, and `vm_service` is the lower-level route. No running app means there's nothing to do: don't start one just to reload it.

## Reload or Restart

| The edit changed | Call | Why |
|---|---|---|
| A `build` method, a widget's layout, or a plain method | `hot_reload` | Injects the new code and keeps state |
| `initState`, a field initializer, or state set up once | `hot_restart` | That code runs only at creation |
| A global, a static or `main()` | `hot_restart` | Reload keeps existing values |
| An enum into a class, a generic type, or a native plugin | Stop the app and run it again | Neither reload path can apply it |

After either call, read `get_runtime_errors`. A reload that reports success can still leave a red screen.
