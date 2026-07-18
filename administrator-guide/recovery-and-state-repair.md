---
title: "Recovery and State Repair"
description: "A conservative method for repairing inconsistent LSS state."
---

# Recovery and State Repair

LSS recovery means restoring agreement between several layers:

```text
Moonbase
→ intended module behavior

/var/state/lunar/packages
→ package and policy state

/var/state/lunar/depends
→ dependency relationships

/var/log/lunar/install
→ final ownership

/var/log/lunar/md5sum
→ installed checksums

filesystem
→ physical payload

runtime
→ actual functionality
```

Repairing only the visible symptom can leave the system inconsistent.

## Preserve evidence first

Before changing anything:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-recovery/$MODULE

mkdir -p "$BASE"

cp -a /var/state/lunar/packages "$BASE/packages.before"
cp -a /var/state/lunar/depends "$BASE/depends.before"

grep "^${MODULE}:" /var/state/lunar/packages \
  > "$BASE/package-record.before" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:" \
  /var/state/lunar/depends \
  > "$BASE/depends-records.before" || true

cp -a /var/log/lunar/install/${MODULE}-* "$BASE/" 2>/dev/null || true
cp -a /var/log/lunar/md5sum/${MODULE}-* "$BASE/" 2>/dev/null || true
cp -a /var/log/lunar/compile/${MODULE}-* "$BASE/" 2>/dev/null || true
```

Also preserve the active module definition, Moonbase revision, configuration choices, and toolchain environment when possible.

## Inconsistency matrix

### Package record present, payload missing

Likely causes:

- manual deletion;
- filesystem damage;
- interrupted replacement;
- restored state database without payload.

Preferred repair:

```text
preserve evidence
→ rebuild or reinstall
→ regenerate payload and logs
→ verify runtime
```

Do not delete the package record merely to make the database resemble the damaged filesystem.

### Package record present, manifest missing

The module is recorded as installed, but LSS has lost its primary ownership record.

Preferred repair:

```text
rebuild or reinstall the same configuration
→ regenerate install and MD5 logs
→ compare the resulting payload
→ verify state
```

Avoid normal removal until ownership has been restored.

### Manifest present, package record missing

Possible causes:

- interrupted removal;
- manual state edit;
- stale log;
- partial recovery.

Inspect:

```bash
grep "$MODULE" /var/log/lunar/activity
```

Then compare the manifest with the actual filesystem. Do not assume either layer is automatically correct.

### Payload present without ownership

Possible causes:

- manual installation;
- preserved configuration;
- partial failed operation;
- external installer;
- files left by a hook or service.

Search existing ownership:

```bash
grep -R -F -x '/path' /var/log/lunar/install 2>/dev/null
```

Do not assign ownership by guesswork.

### Dependency record references a missing module

Inspect the dependent module's `DEPENDS`, `CONFIGURE`, and `OPTIONS` files.

Preferred repair:

```text
reconfigure or rebuild the dependent module
→ regenerate dependency state
→ verify the relationship
```

Deleting one line from `/var/state/lunar/depends` may hide the problem without restoring the intended configuration.

### Preserved configuration remains after removal

This may be correct when a locally modified `/etc` file survives with `PRESERVE=on`.

Such a file is local orphaned state:

```text
present on disk
→ no longer owned by an installed module
```

Keep, archive, compare, or remove it deliberately.

## Drift that affects recovery

Recovery must account for more than files and records.

### Policy drift

Preserve the full state list, including:

```text
held
exiled
enforced
```

Restoring a module as merely `installed` may change administrator intent.

### Moonbase drift

The current Moonbase definition may differ from the one that produced the damaged installation.

Preserve:

```bash
git -C /var/lib/lunar/moonbase rev-parse HEAD
git -C /var/lib/lunar/moonbase diff
```

A rebuild from a changed recipe may repair the module while changing its behavior.

### Optional-feature and provider drift

A reinstall with different options or provider choices can produce a valid but semantically different package.

Inspect saved dependency state and module configuration before recovery.

### Toolchain drift

A new compiler or linker may change ABI, generated code, or build behavior.

Record:

```text
CC
CXX
CFLAGS
CXXFLAGS
LDFLAGS
toolchain versions
```

Use a trusted cache when exact reproduction is required and a matching rebuild environment is unavailable.

## Preferred repair order

Use the least invasive supported path:

```text
1. rebuild
2. reinstall
3. restore through a verified cache
4. rollback from a coherent backup or snapshot
5. narrowly scoped manual repair
```

A supported rebuild or reinstall can restore payload, ownership, checksums, package state, and dependency state together.

## Interrupted rebuild

Possible state:

```text
old payload partly removed
new payload incomplete
final manifest absent
old cache still available
package record stale or missing
```

Response:

1. stop further updates;
2. preserve console, compile, activity, and state evidence;
3. determine whether the old or new payload remains;
4. use a trusted cache or known-good module definition;
5. reinstall;
6. verify ownership and runtime before continuing.

## Interrupted removal

Possible state:

```text
package record removed
some payload remains
module logs partly removed
modified configuration remains
```

Use the surviving manifest, a saved manifest, a trusted cache, or a test reinstall to reconstruct ownership. Avoid broad manual deletion.

## Manual manifest reconstruction

Manual reconstruction is a last resort.

Use it only when:

- rebuild is impossible;
- no trusted cache exists;
- no coherent backup exists;
- ownership must be recovered from strong evidence.

Possible evidence includes:

- a saved package cache;
- an old manifest;
- another identical system;
- upstream installation lists;
- filesystem timestamps;
- activity history.

A guessed manifest can cause destructive removal.

## Manual state repair

Manual editing is a last resort.

Before editing:

```bash
cp -a /var/state/lunar/packages \
  /root/packages.pre-manual-repair

cp -a /var/state/lunar/depends \
  /root/depends.pre-manual-repair
```

A valid manual repair requires:

- a documented source for each reconstructed value;
- one narrowly scoped change;
- preservation of `held`, `exiled`, and `enforced` policy;
- immediate before/after comparison;
- filesystem and runtime verification.

## Recovery checkpoints

After each major repair step:

```bash
cp -a /var/state/lunar/packages \
  "$BASE/packages.checkpoint-N"

cp -a /var/state/lunar/depends \
  "$BASE/depends.checkpoint-N"
```

Do not perform several major repairs without intermediate checkpoints.

## System-wide audit ideas

Check installed records without matching manifests:

```bash
awk -F: '$3 ~ /installed/ {print $1 ":" $4}' \
  /var/state/lunar/packages |
while IFS=: read -r module version; do
  test -f "/var/log/lunar/install/${module}-${version}" ||
    echo "missing manifest: ${module}-${version}"
done
```

Treat audit output as a lead, not automatic proof. Historical or naming exceptions may exist.

Check missing paths in one manifest:

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < /var/log/lunar/install/module-version
```

## Verification after repair

Check package state:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

Check ownership:

```bash
cat /var/log/lunar/install/module-version
```

Check dependency state:

```bash
grep -E '^module:|^[^:]+:module:' \
  /var/state/lunar/depends
```

Check payload and runtime:

```bash
command -v program
ldd /usr/bin/program
program --version
```

A repair is complete only when state, ownership, filesystem, policy, configuration, and runtime agree.

See [Caches and Rollback](caches-and-rollback.html) and [Advanced Inspection](../debugging-guide/advanced-inspection.html).
