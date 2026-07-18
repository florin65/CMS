---
title: "Logs and Manifests"
description: "How to interpret LSS logs, ownership manifests, checksums, and persistent state."
---

# Logs and Manifests

LSS exposes several complementary records.

```text
/var/log/lunar/compile
→ build output

/var/log/lunar/install
→ final ownership manifests

/var/log/lunar/md5sum
→ installed checksums

/var/log/lunar/activity
→ operation history

/var/state/lunar/packages
→ package and policy state

/var/state/lunar/depends
→ dependency state

/var/cache/lunar
→ reusable package archives
```

No single record answers every question.

## Compile logs

Typical location:

```text
/var/log/lunar/compile/module-version.xz
```

Read:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search common errors:

```bash
xzgrep -n -i \
  -E 'error|failed|fatal|undefined|not found|cannot' \
  /var/log/lunar/compile/module-version.xz
```

Find the first meaningful failure, not only the final cascade.

## Install manifests

Typical location:

```text
/var/log/lunar/install/module-version
```

The manifest is the final ownership record used by removal.

```bash
cat /var/log/lunar/install/module-version
```

It may contain:

- regular files;
- symlinks;
- shared directories;
- module-specific directories;
- documentation;
- manual pages;
- Lunar-generated logs.

## MD5 logs

Typical location:

```text
/var/log/lunar/md5sum/module-version
```

They support checksum comparison, especially for `/etc`.

```bash
md5sum /etc/module.conf

grep '/etc/module.conf' \
  /var/log/lunar/md5sum/module-version
```

## Activity log

```text
/var/log/lunar/activity
```

Use it to correlate install, rebuild, and removal history.

```bash
grep 'module' /var/log/lunar/activity
tail -f /var/log/lunar/activity
```

## Persistent state

Package record form:

```text
module:YYYYMMDD:state-list:version:size
```

Dependency record form:

```text
module:dependency:status:type:field5:field6
```

Inspect:

```bash
grep '^module:' /var/state/lunar/packages

grep -E '^module:|^[^:]+:module:' \
  /var/state/lunar/depends
```

## Before and after capture

```bash
MODULE=name
BASE=/root/lss-debug/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

cp -a /var/state/lunar/packages \
  "$BASE/before/packages"

cp -a /var/state/lunar/depends \
  "$BASE/before/depends"

lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"

cp -a /var/state/lunar/packages \
  "$BASE/after/packages"

cp -a /var/state/lunar/depends \
  "$BASE/after/depends"

diff -u "$BASE/before/packages" "$BASE/after/packages"
diff -u "$BASE/before/depends" "$BASE/after/depends"
```

## Manifest versus filesystem

Check missing paths:

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < /var/log/lunar/install/module-version
```

Search ownership of one path:

```bash
grep -R -F -x '/path' \
  /var/log/lunar/install 2>/dev/null
```

Multiple matches may indicate shared or overlapping ownership and require review.

## Troubleshooting order

```text
define the exact failure
→ preserve console output
→ read the first real build error
→ inspect the active Moonbase module
→ inspect package and dependency state
→ compare manifest and filesystem
→ determine whether a cache was used
→ reproduce one controlled operation
→ test runtime behavior
```

Do not edit state databases as a troubleshooting shortcut.
