---
title: "Advanced Inspection"
description: "Controlled evidence collection for difficult LSS problems."
---

# Advanced Inspection

Use advanced inspection only when ordinary logs and manifests cannot answer a precise question.

Good questions include:

```text
Why did installed size change?
Why is a path missing from the final manifest?
Why do two modules appear to own the same path?
Why was a configuration file preserved?
Why did dependency state change?
Was a cache used?
Which operation removed this path?
```

## Preserve before testing

Record:

- package and dependency state;
- install and MD5 logs;
- compile and activity logs;
- cache inventory;
- Moonbase revision and local changes;
- compiler environment;
- filesystem metadata;
- runtime and linkage results.

Use explicit evidence names:

```text
packages.before
packages.after
depends.before
depends.after
install-log.before
install-log.after
console.log
filesystem-state.before
filesystem-state.after
```

Do not overwrite raw evidence.

## Installed-size semantics

Validated behavior:

- installed size is calculated from regular files listed in the final manifest;
- directories are excluded;
- symlinks are excluded;
- Lunar-generated log files listed in the manifest can contribute.

Reproduce:

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    printf '%8s KB  %s\n' "$value" "$path"
    SIZE=$((SIZE + value))
  fi
done < install-log

echo "TOTAL=${SIZE}KB"
```

Use `du -k` explicitly. Interactive aliases such as `du -h` can make manual calculations misleading.

## Raw installwatch activity

Raw installwatch can record events such as:

```text
open
chmod
mkdir
link
symlink
rename
unlink
```

It does not define final ownership.

```text
raw installwatch event
→ something happened

final manifest entry
→ LSS claims final ownership
```

A path may be created and removed during installation and therefore not appear in the final manifest.

## Raw capture safety

Raw installwatch files are normally temporary.

Capture requires controlled instrumentation and should be performed only in a container, chroot, test VM, or other recoverable environment.

Instrument only the destruction boundary, preserve the raw file, and restore LSS source immediately afterward.

## Detecting cache use

Clues include:

- unusually fast completion;
- no normal compiler output;
- no fresh compile log;
- cache timestamp older than the current install;
- activity indicating restoration.

Inspect:

```bash
ls -l /var/cache/lunar/${MODULE}-*
ls -l /var/log/lunar/compile/${MODULE}-*
grep "$MODULE" /var/log/lunar/activity
```

Move the cache aside for one controlled fresh build when necessary.

## Investigating `/etc`

```bash
md5sum /etc/example.conf
grep '/etc/example.conf' saved-md5sum-log
grep -n 'PRESERVE' /etc/lunar/config
```

Interpret checksum, `PRESERVE`, `PROTECTED`, `EXCLUDED`, and remove hooks separately.

## Investigating dependency drift

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:" \
  /var/state/lunar/depends

sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Also inspect `CONFIGURE` and `OPTIONS`.

Possible causes:

- old stored choice;
- changed Moonbase definition;
- interrupted reconfiguration;
- renamed provider;
- changed optional-feature default.

## Single-variable experiments

Change one factor only:

```text
compiler
dependency choice
cache availability
Moonbase revision
PRESERVE setting
module patch
```

Changing several factors at once destroys attribution.

## Escalation ladder

```text
1. lvu and state files
2. compile, install, MD5, and activity logs
3. Moonbase source inspection
4. before/after snapshots
5. cache isolation
6. container reproduction
7. raw installwatch capture
8. source-level instrumentation
```

Stop when the question is answered.
