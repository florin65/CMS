---
title: "Lunar Script System Documentation"
description: "Documentation for using, administering, extending, debugging, and recovering Lunar Linux systems with LSS."
status: "Draft 0.4"
---

# Lunar Script System Documentation

The Lunar Script System (LSS) is Lunar Linux's source-based package management and software build system.

LSS uses Moonbase modules to describe how software is obtained, verified, configured, built, installed, rebuilt, and removed. It also records package state, dependency relationships, ownership manifests, checksums, compile logs, activity history, and reusable package caches.

## Operating model

Use the same sequence for routine administration and troubleshooting:

```text
understand module intent
→ inspect current state
→ perform one controlled operation
→ verify ownership and persistent state
→ verify the filesystem and runtime result
```

The main information layers are:

```text
Moonbase
→ module intent and build policy

compile logs and raw installwatch activity
→ observed execution

install manifest
→ final package ownership

packages and depends
→ persistent package and dependency state

filesystem and runtime
→ the actual system result
```

No single layer answers every question.

```text
raw installwatch event
→ something happened

final manifest entry
→ LSS claims final ownership
```

## Documentation map

### [User Guide](user-guide/index.md)

Normal installation, rebuild, removal, dependencies, configuration, policy states, inspection, and system updates.

### [Administrator and Recovery Guide](administrator-guide/index.md)

Configuration preservation, rollback, inconsistent state, and conservative system repair.

### [Module Author Guide](module-author-guide/index.md)

Moonbase module structure, lifecycle boundaries, hooks, dependencies, and validation.

### [Debugging and Inspection Guide](debugging-guide/index.md)

Logs, manifests, raw installwatch activity, installed-size analysis, ownership questions, and advanced evidence collection.

### [Plugin and Extension Guide](plugin-guide/index.md)

Plugin control flow, ordering, reload behavior, testing, and extension safety.

### [Command and State Reference](reference/index.md)

Commands, paths, record formats, state semantics, and plugin return values.

## Information-status labels

Some implementation details are marked when the distinction matters:

```text
Validated behavior
→ confirmed by runtime observation

Source-derived behavior
→ established from current LSS source code

Recommended practice
→ operational guidance
```

The public documentation is limited to Lunar Linux, LSS, Moonbase, and directly relevant technical behavior.
