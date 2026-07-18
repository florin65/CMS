---
title: "Configuration and Reconfiguration"
description: "How module options, stored choices, and system configuration interact."
---

# Configuration and Reconfiguration

LSS configuration exists at several levels:

```text
/etc/lunar/config
→ global LSS behavior

Moonbase CONFIGURE and OPTIONS
→ module choices

/var/state/lunar/depends
→ resolved dependency choices

/etc
→ installed runtime configuration
```

Do not collapse these into one concept.

## Inspect global settings

```bash
grep -n -E 'ARCHIVE|PRESERVE|REAP' \
  /etc/lunar/config
```

Common meanings:

```text
ARCHIVE
→ create reusable package archives

PRESERVE
→ control treatment of locally modified `/etc` files

REAP
→ control cleanup behavior
```

## Inspect module choices

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

for file in CONFIGURE OPTIONS DEPENDS BUILD; do
  test -f "$MODULE_DIR/$file" || continue
  echo "=== $file ==="
  sed -n '1,240p' "$MODULE_DIR/$file"
done
```

## Reconfiguration

A choice does not change the installed binary until the module is rebuilt.

Safe process:

```text
inspect old choices and dependencies
→ preserve state and manifest
→ change one option
→ rebuild
→ inspect dependency and ownership changes
→ test the affected feature
```

## Configuration drift

A Moonbase update can:

- add a new option;
- remove an old option;
- change defaults;
- rename a provider;
- change auto-detection;
- turn an optional dependency into a required one.

Stored choices may therefore represent an older module definition.

## `/etc` files

Before a risky rebuild or removal:

```bash
cp -a /etc/module.conf \
  /root/module.conf.before
```

Afterward:

```bash
diff -u \
  /root/module.conf.before \
  /etc/module.conf
```

Validated behavior with `PRESERVE=on`:

```text
unchanged configuration
→ removed during normal removal

locally modified configuration
→ retained
```

A service update is complete only after configuration and runtime behavior have been tested.
