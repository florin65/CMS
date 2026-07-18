---
title: "Lifecycle and prepare_install"
description: "The installation-tracking boundary and module lifecycle responsibilities."
---

# Lifecycle and prepare_install

A Moonbase module participates in a larger LSS transaction.

A simplified lifecycle is:

```text
preconditions and configuration
→ dependency resolution
→ source download
→ source preparation
→ build
→ prepare_install
→ install into the live filesystem
→ final manifest and MD5 generation
→ package-state update
→ post-install work
```

## The `prepare_install` boundary

`prepare_install` establishes the installation-tracking context before package payload is written into the live filesystem.

Conceptually it:

```text
prepares upgrade-style removal of old ownership
→ activates installwatch
→ establishes the tracked installation phase
```

The exact implementation belongs to LSS, but the module-author responsibility is simple:

```text
build first
→ call prepare_install
→ install second
```

A module that installs files before this boundary can escape normal final ownership tracking.

## Minimal BUILD examples

Conventional Autotools-style project:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

Simple Makefile project:

```bash
make &&
prepare_install &&
make install PREFIX=/usr
```

The installation command and arguments vary by project. The ordering does not.

## Build tree versus live filesystem

Before `prepare_install`:

```text
source and build-tree activity
```

After `prepare_install`:

```text
live-filesystem installation observed by LSS
```

Temporary files in the build tree should not become package ownership.

## Final ownership

Three layers must remain distinct:

```text
Moonbase module
→ intended behavior

raw installwatch activity
→ filesystem events during execution

install manifest
→ final paths attributed to the module
```

A path may appear in raw activity and disappear before the final manifest is created.

## Hooks

Typical hooks include:

```text
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

Use hooks only for lifecycle work that genuinely belongs to the package.

Good hook behavior:

- narrow scope;
- clear failure handling;
- predictable paths;
- no hidden global changes;
- no avoidable writes outside tracked ownership;
- useful diagnostic output.

A module hook is appropriate for package-specific behavior. Reusable LSS-wide interception belongs in the plugin system.

## Configuration

Files under `/etc` require deliberate handling.

A module author should know:

- which default files are installed;
- whether upstream overwrites them;
- how LSS MD5 tracking will see them;
- whether remove hooks touch them;
- whether local modification should survive removal.

Do not install generated credentials or host-specific secrets as normal package defaults.

## Dependencies

Declare real build and runtime relationships.

Do not rely on software that happens to be present on the development host.

Optional dependencies should map clearly to features and should use explicit positive and negative build flags where possible.

## Author acceptance criteria

A lifecycle is acceptable when:

```text
source verifies
build succeeds
prepare_install precedes live installation
final manifest contains intended ownership
package state is correct
dependencies are correct
runtime works
rebuild is stable
removal is clean
reinstall works
```
