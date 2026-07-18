---
title: "Caches and Rollback"
description: "How LSS caches help recovery and where their limits begin."
---

# Caches and Rollback

LSS caches under:

```text
/var/cache/lunar
```

can accelerate reinstall and rollback.

A typical cache name may include:

```text
module-version-target-triplet.tar.xz
```

Example:

```text
xxhash-0.8.3-x86_64-pc-linux-gnu.tar.xz
```

## What a cache is

A cache is reusable installed payload.

It is not:

- proof that the module is currently installed;
- the package-state record;
- the dependency-state record;
- the final ownership manifest;
- a complete description of build configuration;
- a guarantee that post-install effects will be reproduced.

```text
cache present
≠ installed

installed
≠ cache present
```

## Cache identity is incomplete

A matching filename may still hide different builds.

```text
matching module + version + target triplet
≠ guaranteed semantic equivalence
```

Two caches may differ because of:

- optional dependencies;
- provider selection;
- compiler family or version;
- compiler flags;
- patches;
- Moonbase revision;
- local environment;
- generated configuration.

Preserve provenance for important caches.

## Inspecting a cache

List:

```bash
ls -lh /var/cache/lunar/module-*
```

Test compression:

```bash
xz -t archive.tar.xz
```

List contents:

```bash
tar -tJf archive.tar.xz
```

Extract only into a temporary directory for inspection:

```bash
mkdir -p /tmp/lunar-cache-inspect

tar -xJf archive.tar.xz \
  -C /tmp/lunar-cache-inspect
```

Do not extract directly into `/`.

## Preserving a trusted cache

```bash
mkdir -p /root/lunar-cache-backup

cp -a /var/cache/lunar/module-version-* \
  /root/lunar-cache-backup/

sha256sum /root/lunar-cache-backup/module-version-* \
  > /root/lunar-cache-backup/SHA256SUMS
```

Useful sidecar metadata includes:

```text
module
version
creation date
Moonbase revision
compiler versions
build flags
optional dependency choices
target triplet
checksum
```

## Before risky work

Preserve:

- a known-good cache;
- `/var/state/lunar/packages`;
- `/var/state/lunar/depends`;
- install and MD5 logs;
- active Moonbase revision and local changes;
- critical configuration;
- a tested boot or recovery path.

## Fresh rebuild versus cache resurrection

A cache can hide the normal compile path.

When a fresh build is required:

```text
preserve old cache
→ move it outside /var/cache/lunar
→ rebuild
→ inspect new manifest and runtime
→ preserve the new cache
```

A fresh build is preferable when:

- toolchain changed;
- Moonbase recipe changed;
- optional features changed;
- provider changed;
- patches changed;
- reproducibility is under test.

## Rollback completeness

A useful rollback restores:

```text
payload
final ownership
MD5 records
package record
dependency relationships
policy state
configuration
runtime behavior
```

Restoring only binaries is incomplete.

Restoring only state databases is also incomplete.

## Recovery through LSS

Use normal LSS installation or rebuild paths whenever possible.

```text
trusted cache
→ restore through LSS
→ regenerate state
→ verify ownership
→ verify runtime
```

Manual unpacking can bypass:

- manifest generation;
- package registration;
- dependency-state updates;
- configuration policy;
- post-install behavior.

## Snapshot and backup strategy

For experiments:

```text
backup or snapshot
→ perform one controlled transition
→ preserve evidence
→ restore if necessary
```

For full-system recovery, preserve both files and state:

```text
/etc
/var/state/lunar
/var/log/lunar
/var/cache/lunar
active Moonbase
important source archives
```

A backup that has never been tested is only a hypothesis.
