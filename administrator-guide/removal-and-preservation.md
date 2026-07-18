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

Module-specific logs may disappear during removal.

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

The `PRESERVE=off` path was not runtime-validated in the current documented evidence. Treat its exact behavior as implementation-derived until tested.

## Normal removal versus upgrade removal

A rebuild may use upgrade-style removal to replace the old package while preparing the new installation.

```text
normal lrm
→ user-requested removal

upgrade removal
→ replacement phase inside rebuild or update
```

Do not assume every detail of normal removal is identical during upgrade replacement.

## Removal after a failed installation

A failed install may leave:

- partial payload;
- no package record;
- stale package record;
- incomplete manifest;
- temporary files.

Inspect ownership evidence before running `lrm`.

If the final manifest is incomplete, normal removal may not clean all partial files.

## Package record present, manifest missing

Normal removal is unsafe because ownership evidence is incomplete.

Preferred path:

```text
preserve state
→ rebuild or reinstall the same configuration
→ regenerate manifest
→ verify
→ remove if still desired
```

## Manifest present, package record missing

Possible causes include:

- interrupted removal;
- manual state edit;
- stale logs.

Inspect the activity log and actual payload before deciding whether the manifest is stale.

## Shared ownership

Before deleting a suspicious remaining path:

```bash
grep -R -F -x '/path' \
  /var/log/lunar/install 2>/dev/null
```

Multiple matches may indicate intentional sharing, a provider transition, stale state, or a packaging error.

## Files recreated after removal

A path may reappear because:

- a service recreates it;
- another module owns it;
- a login or boot process generates it;
- a hook rebuilds it;
- an index or cache refresh recreates it.

Compare timestamps and inspect service and hook behavior before assuming removal failed.

## Services and active processes

Before removing a service:

- stop it;
- inspect active processes;
- inspect listening sockets;
- preserve configuration;
- inspect service-manager files;
- verify dependent services.

Removing files does not terminate an already running process.

## Plugin-bearing modules

A module may install LSS plugin files.

Before removal:

- identify plugin ownership;
- wait for active `lin` operations to finish;
- inspect fallback behavior;
- preserve plugin configuration;
- verify plugin directories afterward.

A running process may already have loaded plugin code even after the file disappears.

See [Plugins and Extensibility](../plugin-guide/plugins-and-extensibility.html).

## Critical modules

Do not casually remove:

```text
lunar
glibc
gcc
bash
coreutils
filesystem components
init system
boot components
core shared libraries
```

Sparse dependency state does not prove operational safety.

Use a container, chroot, or recovery environment for experiments involving critical modules.

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
- activity history;
- recreated paths.

Do not treat the presence of a preserved configuration file as proof that the module remains installed.

For inconsistent removal state, see [Recovery and State Repair](recovery-and-state-repair.html).
