---
title: "Updating the System Safely"
description: "A staged method for updating modules, libraries, toolchains, and services."
---

# Updating the System Safely

An update can change:

- version;
- dependencies;
- ABI;
- compiler output;
- installed paths;
- configuration defaults;
- provider selection;
- service behavior.

Use staged updates rather than one undifferentiated system change.

## Before updating Moonbase

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase rev-parse HEAD \
  > /root/moonbase-revision.before

git -C /var/lib/lunar/moonbase diff \
  > /root/moonbase-local-changes.patch
```

Preserve package and dependency state:

```bash
cp -a /var/state/lunar/packages \
  /root/packages.before-update

cp -a /var/state/lunar/depends \
  /root/depends.before-update
```

## Review changed modules

Compare the old and new Moonbase revisions:

```bash
git -C /var/lib/lunar/moonbase diff \
  OLD..NEW -- section/module
```

Look for:

- version and source changes;
- checksum changes;
- new required dependencies;
- optional-feature changes;
- compiler selection;
- install path changes;
- new hooks;
- removal behavior.

The same upstream version can still have different packaging behavior.

## Update order

```text
dependencies
→ shared libraries
→ direct consumers
→ applications and services
```

A shared-library update may require rebuilding reverse dependents even when their own versions do not change.

Check:

```bash
lvu depends library
lvu leert library
```

## Toolchain updates

Record:

```bash
gcc --version
clang --version
ld --version
make --version

env | grep -E \
  '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS)='
```

Validate the new toolchain on a small known module before rebuilding core components.

Compiler-family consistency matters. A module expecting Clang-specific flags should not silently build with GCC, and vice versa.

## Update waves

A practical order:

```text
wave 1
→ small isolated utilities

wave 2
→ common libraries

wave 3
→ applications

wave 4
→ services

wave 5
→ core and toolchain
```

Capture `/var/state/lunar/packages` and `/var/state/lunar/depends` after each major wave.

## Service updates

After updating a service:

```text
verify binary
→ compare configuration
→ restart or reload
→ inspect logs
→ test client behavior
```

Installation success is not service validation.

## Failure response

```text
stop expansion
→ preserve evidence
→ identify failed lifecycle phase
→ inspect package and ownership state
→ restore from trusted cache or rebuild
→ verify before continuing
```

Do not continue a large update after an unexplained core failure.

## Final verification

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
cat /var/log/lunar/install/module-version
ldd /usr/bin/program | grep 'not found'
```

Test the actual feature or service.
