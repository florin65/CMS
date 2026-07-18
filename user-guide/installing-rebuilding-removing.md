---
title: "Installing, Rebuilding, and Removing Modules"
description: "Safe daily module operations with lin, lrm, and lvu."
---

# Installing, Rebuilding, and Removing Modules

The core LSS operations are:

```bash
lin module
lin -c module
lrm module
```

Use `lvu` before and after each operation to inspect state.

## Before installation

Locate and inspect the module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf 'section: %s\n' "$SECTION"

sed -n '1,240p' "$MODULE_DIR/DETAILS"
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Check:

- source and version;
- required dependencies;
- optional features;
- conflicts;
- unusual hooks;
- local Moonbase changes.

## Install

```bash
lin "$MODULE"
```

Preserve console output for important modules:

```bash
lin "$MODULE" 2>&1 |
  tee "/root/${MODULE}-install.log"
```

Verify:

```bash
lvu installed "$MODULE"
grep "^${MODULE}:" /var/state/lunar/packages
```

Inspect ownership:

```bash
VERSION=$(lvu installed "$MODULE")
cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

Then test actual functionality.

A successful `lin` result proves that the package transaction completed. It does not prove that every application feature or service works.

## Rebuild

```bash
lin -c "$MODULE"
```

Use rebuild when:

- changing options;
- changing compiler or flags;
- rebuilding after a dependency transition;
- testing a Moonbase modification;
- refreshing a damaged installation;
- validating reproducibility.

Before rebuilding, preserve the current package record and manifest:

```bash
VERSION=$(lvu installed "$MODULE")

grep "^${MODULE}:" /var/state/lunar/packages \
  > "/root/${MODULE}.package.before"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}" \
  "/root/${MODULE}.manifest.before"
```

After rebuilding:

```bash
VERSION_AFTER=$(lvu installed "$MODULE")

grep "^${MODULE}:" /var/state/lunar/packages \
  > "/root/${MODULE}.package.after"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}" \
  "/root/${MODULE}.manifest.after"

diff -u \
  "/root/${MODULE}.manifest.before" \
  "/root/${MODULE}.manifest.after"
```

A changed manifest may be expected after a version or feature change. It should still be understood.

## Before removal

Check reverse dependencies:

```bash
lvu depends "$MODULE"
lvu leert "$MODULE"

grep -E "^[^:]+:${MODULE}:" \
  /var/state/lunar/depends
```

Inspect remove hooks:

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  echo "=== $hook ==="
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

Preserve install and MD5 logs because removal may delete them:

```bash
VERSION=$(lvu installed "$MODULE")
BACKUP="/root/${MODULE}-remove-evidence"

mkdir -p "$BACKUP"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}" \
  "$BACKUP/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}" \
  "$BACKUP/md5sum-log" 2>/dev/null || true
```

## Remove

```bash
lrm "$MODULE"
```

Verify:

```bash
grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "package record removed"
```

Use the saved manifest to confirm payload removal.

Shared directories such as `/usr` and `/usr/share` should not be removed simply because they appeared in a module manifest.

## Configuration behavior

With `PRESERVE=on`, validated behavior is:

```text
unchanged `/etc` file
→ removed

locally modified `/etc` file
→ retained
```

A retained file after package removal is local orphaned configuration. It is no longer evidence that the package remains installed.

## Safe daily pattern

```text
inspect
→ preserve current state
→ perform one operation
→ verify package record
→ verify manifest
→ verify runtime
```
