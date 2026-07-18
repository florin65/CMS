---
title: "Dependencies and Optional Features"
description: "How LSS records dependencies and how to change optional module features safely."
---

# Dependencies and Optional Features

Moonbase modules can declare required and optional relationships.

## Required dependencies

A required dependency belongs to the module's accepted build or runtime form.

Inspect:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Before removing a dependency, inspect reverse use:

```bash
lvu depends dependency
lvu leert dependency

grep -E '^[^:]+:dependency:' \
  /var/state/lunar/depends
```

## Optional dependencies

Optional dependencies normally control features.

Conceptually:

```text
dependency enabled
→ feature enabled
→ build flags and stored relationship change

dependency disabled
→ feature omitted
→ negative build flag should prevent accidental detection
```

The exact prompts and flags are module-specific.

## Current resolved state

```bash
grep "^${MODULE}:" /var/state/lunar/depends
```

Observed form:

```text
module:dependency:status:type:field5:field6
```

This is persistent resolved state, not merely a copy of the current `DEPENDS` file.

Moonbase may change while stored state still reflects an earlier configuration.

## Changing an optional feature

A safe sequence:

```text
inspect current DEPENDS and stored state
→ preserve manifest and dependency records
→ reconfigure or rebuild
→ inspect new dependency state
→ compare manifest
→ test the feature
```

Example evidence capture:

```bash
grep "^${MODULE}:" /var/state/lunar/depends \
  > "/root/${MODULE}.depends.before"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}" \
  "/root/${MODULE}.manifest.before"

lin -c "$MODULE"

grep "^${MODULE}:" /var/state/lunar/depends \
  > "/root/${MODULE}.depends.after"
```

## Removing an optional dependency

Do not remove the dependency first.

Correct order:

```text
disable feature in dependent module
→ rebuild dependent
→ verify relationship disappeared
→ verify runtime
→ remove former dependency
```

Otherwise the current installed binary may still require the library even if the planned next configuration does not.

## Provider changes

When several implementations can satisfy a role:

```text
select provider
→ rebuild dependent modules
→ verify stored relationships
→ verify runtime
→ remove old provider only after transition
```

Policy states such as `exiled` and `enforced` may also express provider intent.

## Hidden auto-detection

Upstream configure systems may detect installed software automatically.

Module definitions should use explicit positive and negative flags where possible. Users should verify that disabling a dependency really removes the feature from the build.
