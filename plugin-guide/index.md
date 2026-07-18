---
title: "LSS Plugin and Extension Guide"
description: "Plugin control flow, ordering, reload behavior, and safe extension practices."
---

# LSS Plugin and Extension Guide

Plugins extend LSS lifecycle behavior and run with package-management authority.

## Return-value contract

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ not handled; continue
```

This is a control-flow contract, not only a shell success/failure convention.

## Chaining example

```text
plugin A returns 2
→ plugin B runs

plugin B returns 0
→ chain stops successfully

default implementation
→ not called
```

Failure case:

```text
plugin A returns 1
→ chain stops as failure
```

Fallback case:

```text
all plugins return 2
→ default LSS implementation may run
```

## Ordering

Registration order matters.

An earlier general plugin can handle an operation and prevent a later specialized plugin from running.

A plugin that appears inactive may be shadowed rather than broken.

## Installation and reload

Modules may install plugin files.

A running LSS process may already have loaded plugin code even after the file is replaced or removed.

Source-derived behavior: current source includes `USR1`-based plugin reload behavior.

Treat reload as advanced and implementation-specific. Verify the active LSS version before relying on it.

Do not change plugin files during active package operations.

## Safe plugin design

A plugin should:

- have one narrow responsibility;
- return `2` when it does not handle a case;
- preserve LSS error semantics;
- quote paths and data safely;
- avoid untracked payload writes;
- avoid undocumented dependence on mutable core globals;
- leave useful diagnostic evidence;
- preserve ownership and state expectations.

Returning `0` means the plugin accepts responsibility for successful completion.

## Test matrix

A plugin is incomplete until all paths are tested.

### Success

```text
eligible case
→ plugin handles operation
→ returns 0
→ expected state and runtime result exist
```

### Failure

```text
eligible case
→ plugin cannot complete safely
→ returns 1
→ chain stops
→ failure evidence is clear
```

### Not handled

```text
ineligible case
→ plugin returns 2
→ next plugin or default implementation runs
```

## Ownership warning

If a plugin writes files outside normal tracked installation, those paths may not appear in a module manifest.

A plugin that changes installation behavior must preserve LSS ownership guarantees or document the boundary explicitly.

## Removal safety

Before removing a module that installed a plugin:

1. identify active `lin` processes;
2. identify the plugin file and owning manifest;
3. inspect fallback behavior;
4. allow active operations to finish;
5. remove the module;
6. verify the plugin directory;
7. run a low-risk LSS command.

## Hook or plugin?

Use a module hook for package-specific lifecycle work.

Use a plugin for reusable LSS-wide interception.

Avoid moving fundamental package-state correctness into optional plugin behavior.
