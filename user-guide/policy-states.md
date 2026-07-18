---
title: "Held, Exiled, and Enforced Modules"
description: "How LSS preserves administrator policy in package state."
---

# Held, Exiled, and Enforced Modules

The package-state record can preserve administrative policy.

```text
installed
→ module is recorded as present

held
→ retain the current version

exiled
→ reject module presence

enforced
→ require module presence
```

These states are not interchangeable.

## Held

Use a hold when a known-good version must remain in place.

Typical reasons:

- regression in a newer version;
- dependent software not ready;
- local patch;
- incomplete toolchain transition;
- production validation still pending.

Document why the hold exists and what condition will end it.

A long-lived hold can create security and compatibility debt.

## Exiled

Exile is stronger than simply not installing a module.

```text
uninstalled
→ absent now

exiled
→ explicitly rejected by policy
```

Exiling a dependency can make another module impossible to install or rebuild.

Review reverse relationships first.

## Enforced

An enforced module is required by system policy even when no other module depends on it.

```text
required dependency
→ required by another module

enforced module
→ required by administrator policy
```

Do not remove an enforced module without deliberately changing the policy.

## Inspect policy

```bash
awk -F: '$3 ~ /held|exiled|enforced/ {print}' \
  /var/state/lunar/packages
```

For one module:

```bash
grep '^module:' /var/state/lunar/packages
lvu depends module
lvu leert module
```

## Before large updates

Review all special states.

A held module may block an ABI transition. An exiled module may conflict with a new required dependency. An enforced provider may affect replacement decisions.

Preserve complete package records during rollback. Restoring only `installed` can silently lose administrator intent.
