---
title: "Removal and Configuration Preservation"
description: "Ownership-driven removal, shared directories, and local configuration."
---

# Removal and Configuration Preservation

LSS removal is driven primarily by the final install manifest.

```text
manifest path
→ classify
→ apply path and configuration policy
→ remove when permitted
→ update persistent state
```

## Before removal

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Check reverse dependencies:

```bash
lvu depends "$MODULE"
lvu leert "$MODULE"
```

Inspect remove hooks:

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

Save logs:

```bash
mkdir -p "/root/${MODULE}-remove"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}" \
  "/root/${MODULE}-remove/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}" \
  "/root/${MODULE}-remove/md5sum-log" 2>/dev/null || true
```

## Regular files and symlinks

Owned payload is normally removed.

Symlink ownership and target ownership are separate. Removing an owned symlink does not imply that its target is owned by the same module.

## Directories

Shared directories are not removed blindly.

```text
shared non-empty directory
→ retained

module-specific empty directory
→ may be removed
```

## Configuration policy

```text
PRESERVE
→ modified `/etc` configuration may survive removal

PROTECTED
→ ownership may remain recorded, but deletion is blocked

EXCLUDED
→ path is filtered from final ownership
```

Validated behavior with `PRESERVE=on`:

```text
unchanged `/etc` file
→ removed

locally modified `/etc` file
→ retained
```

The retained file becomes local orphaned configuration after the package record and manifest are removed.

## Shared ownership

Before deleting a suspicious remaining path:

```bash
grep -R -F -x '/path' \
  /var/log/lunar/install 2>/dev/null
```

Multiple matches may indicate intentional sharing, a provider transition, stale state, or a packaging error.

## After removal

Verify:

```bash
grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "package record removed"
```

Use the saved manifest to inspect the filesystem.

Also verify:

- dependent services;
- plugin files;
- preserved configuration;
- shared directories;
- activity history.

Do not treat the presence of a preserved configuration file as proof that the module remains installed.
