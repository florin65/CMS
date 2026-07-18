---
title: "LSS Commands and Paths"
description: "A compact reference for common LSS commands and storage locations."
---

# LSS Commands and Paths

## Core commands

Install:

```bash
lin module
```

Rebuild:

```bash
lin -c module
```

Remove:

```bash
lrm module
```

Inspect:

```bash
lvu installed module
lvu version module
lvu where module
lvu depends module
lvu leert module
lvu leafs
```

## Command meanings

```text
lin module
→ install a module

lin -c module
→ rebuild through upgrade-style replacement

lrm module
→ remove through final ownership

lvu installed module
→ show installed version

lvu version module
→ show active Moonbase version

lvu where module
→ show Moonbase section

lvu depends module
→ show reverse dependents

lvu leert module
→ show reverse dependency tree

lvu leafs
→ list modules with no recorded reverse dependents
```

A leaf is not automatically unused or safe to remove.

## Moonbase

Active tree:

```text
/var/lib/lunar/moonbase
```

Locate a module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Common files:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

## Persistent state

```text
/var/state/lunar/packages
/var/state/lunar/depends
```

Package lookup:

```bash
grep '^module:' /var/state/lunar/packages
```

Dependency lookup:

```bash
grep -E '^module:|^[^:]+:module:' \
  /var/state/lunar/depends
```

## Logs and ownership

```text
/var/log/lunar/install
→ final ownership

/var/log/lunar/compile
→ build output

/var/log/lunar/md5sum
→ installed checksums

/var/log/lunar/activity
→ operation history
```

Read a compile log:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

## Caches

```text
/var/cache/lunar
```

Inspect:

```bash
ls -lh /var/cache/lunar/module-*
tar -tJf archive.tar.xz
xz -t archive.tar.xz
```

## Global configuration

```text
/etc/lunar/config
```

Common settings:

```text
ARCHIVE
PRESERVE
REAP
```

Inspect:

```bash
grep -n -E 'ARCHIVE|PRESERVE|REAP' \
  /etc/lunar/config
```

## Useful checks

Runtime:

```bash
command -v program
ldd /path/to/binary
```

Pkg-config:

```bash
pkg-config --modversion name
pkg-config --print-requires name
```

Ownership:

```bash
grep -R -F -x '/path' \
  /var/log/lunar/install 2>/dev/null
```

Moonbase revision:

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase rev-parse HEAD
```

Configuration checksum:

```bash
md5sum /etc/module.conf
```

## Essential paths

```text
/sbin/lin
/sbin/lrm
/bin/lvu
/etc/lunar
/var/lib/lunar/functions
/var/lib/lunar/moonbase
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/activity
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
/var/cache/lunar
/usr/lib/installwatch.so
```
