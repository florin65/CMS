---
title: "Building and Testing a Local Module"
description: "A complete lifecycle test for local Moonbase modules."
---

# Building and Testing a Local Module

Use a container, chroot, virtual machine, or other recoverable environment.

A first test module should be small:

- one source archive;
- conventional build system;
- few dependencies;
- no boot or login integration;
- small, obvious payload;
- simple runtime test.

## Minimal structure

```text
hello-lunar/
├── DETAILS
└── BUILD
```

Add these only when needed:

```text
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
patches
auxiliary files
```

## Minimal `DETAILS` example

```bash
MODULE=hello-lunar
VERSION=1.0
SOURCE=$MODULE-$VERSION.tar.gz
SOURCE_URL=https://example.org/releases/
SOURCE_VFY=sha256:...
WEB_SITE=https://example.org/
ENTERED=20260718
UPDATED=20260718
SHORT="Small example program"

cat << EOF
A small example program packaged as a local Moonbase module.
EOF
```

Use the verification syntax accepted by the active Moonbase.

Calculate a checksum:

```bash
sha256sum hello-lunar-1.0.tar.gz
```

## Minimal `BUILD` example

```bash
make &&
prepare_install &&
make install PREFIX=/usr
```

For a configure-based project:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

## Record the baseline

```bash
MODULE=hello-lunar
BASE=/root/lss-local-module/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

cp -a /var/state/lunar/packages \
  "$BASE/before/packages"

cp -a /var/state/lunar/depends \
  "$BASE/before/depends"

env | sort > "$BASE/before/environment"
```

Also preserve the active Moonbase revision and the module directory.

## Install test

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/install-console.log"
```

Verify:

```bash
lvu installed "$MODULE"
grep "^${MODULE}:" /var/state/lunar/packages
```

Inspect the manifest:

```bash
VERSION=$(lvu installed "$MODULE")
MANIFEST="/var/log/lunar/install/${MODULE}-${VERSION}"

cat "$MANIFEST"
```

Check that it contains only intended paths.

## Runtime test

For a command:

```bash
command -v hello-lunar
hello-lunar --version
hello-lunar
```

Check linkage:

```bash
ldd /usr/bin/hello-lunar
```

Declared dependencies should match actual runtime needs.

## Rebuild test

```bash
cp -a "$MANIFEST" "$BASE/install.before-rebuild"

lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"

cp -a "$MANIFEST" "$BASE/install.after-rebuild"

diff -u \
  "$BASE/install.before-rebuild" \
  "$BASE/install.after-rebuild"
```

A deterministic module should normally produce stable ownership.

## Removal test

Preserve the manifest and MD5 log before removal because module-specific logs may be deleted.

```bash
cp -a "$MANIFEST" "$BASE/install-log"
cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}" \
  "$BASE/md5sum-log" 2>/dev/null || true

lrm "$MODULE" 2>&1 |
  tee "$BASE/remove-console.log"
```

Verify:

- package record is gone;
- payload is gone;
- shared directories remain;
- module-specific empty directories disappear;
- configuration behavior is understood.

## Reinstall test

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/reinstall-console.log"
```

Run the runtime test again.

## Optional dependency test

For one optional feature:

```text
enable feature
→ build
→ record depends and manifest
→ disable feature
→ rebuild
→ compare depends, manifest, and runtime
```

Do not test several option changes at once.

## Failure-path test

Test at least one expected failure:

- unavailable source;
- wrong checksum;
- missing required dependency;
- intentionally failing patch;
- invalid compiler selection.

Verify that failure leaves:

- no false installed record;
- no unmanaged live payload;
- a useful compile log;
- a recoverable next run.

## Hidden dependency detection

A minimal environment reveals undeclared assumptions.

Look for:

```text
command not found
header missing
library missing
pkg-config package missing
```

Add real dependencies rather than depending on the developer's full system.

## Final acceptance

A local module is ready for wider use when:

```text
source verifies
build succeeds
manifest is correct
package state is correct
dependencies are correct
runtime works
rebuild is stable
removal is clean
reinstall works
failure behavior is understandable
```
