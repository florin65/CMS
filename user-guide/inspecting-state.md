---
title: "Inspecting Modules and System State"
description: "How to inspect installed versions, dependencies, ownership, and physical state."
---

# Inspecting Modules and System State

Use several views together.

## Module identity

```bash
lvu installed module
lvu version module
lvu where module
```

These answer:

```text
installed
→ which version is recorded locally?

version
→ which version is available in active Moonbase?

where
→ which Moonbase section contains the definition?
```

## Package state

```bash
grep '^module:' /var/state/lunar/packages
```

Observed form:

```text
module:YYYYMMDD:state-list:version:size
```

Do not read the state field as only installed or absent. It may also preserve policy such as `held`, `exiled`, or `enforced`.

## Dependency state

```bash
grep '^module:' /var/state/lunar/depends

grep -E '^[^:]+:module:' \
  /var/state/lunar/depends
```

The first shows relationships declared for the module. The second shows modules that record it as a dependency.

Use:

```bash
lvu depends module
lvu leert module
```

for a more readable reverse view.

## Ownership

```bash
cat /var/log/lunar/install/module-version
```

The install manifest records final ownership.

Check whether a path is owned:

```bash
grep -R -F -x '/path' \
  /var/log/lunar/install 2>/dev/null
```

Multiple matches require review.

## Checksums

```bash
cat /var/log/lunar/md5sum/module-version
```

For configuration:

```bash
md5sum /etc/module.conf

grep '/etc/module.conf' \
  /var/log/lunar/md5sum/module-version
```

## Physical state

Persistent records can drift from the filesystem.

Check a binary:

```bash
command -v program
file /usr/bin/program
ldd /usr/bin/program
```

Check all paths in one manifest:

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < /var/log/lunar/install/module-version
```

## Interpreting disagreement

```text
package record present + payload missing
→ damaged or incomplete installation

manifest present + package record missing
→ stale or interrupted state

payload present + no manifest
→ unmanaged or orphaned files

Moonbase newer + installed version older
→ pending update, not necessarily damage
```

Inspect before repairing.
