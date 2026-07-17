---
title: "Lunar Script System User Guide"
description: "A practical guide to installing, rebuilding, inspecting, updating, debugging, and recovering software with Lunar Linux's Lunar Script System."
version: "0.1"
date: "2026-07-17"
status: "Publication draft"
---

# Lunar Script System User Guide

The Lunar Script System, or LSS, is the package management and software build system used by Lunar Linux.

LSS combines source-based software construction with explicit module definitions, dependency tracking, installation ownership, configuration preservation, caches, logs, and persistent package state. This guide explains how those parts work together and how to operate them safely.

The guide is written for users and system administrators who want to understand not only which command to run, but also what LSS changes, what evidence it preserves, and how to verify the result.

> **Administrative note**
>
> Most installation, rebuild, removal, and repair operations require root privileges. Test high-risk changes in a container, chroot, virtual machine, or other recoverable environment before applying them to an important system.

## Operating model

LSS administration follows one consistent sequence:

```text
understand intent
→ inspect current state
→ preserve evidence
→ perform one controlled operation
→ verify ownership
→ verify persistent state
→ verify runtime behavior
```

The most important state layers are:

```text
Moonbase
→ module intent and build policy

installwatch and logs
→ observed execution

install manifest
→ final ownership

packages and depends
→ persistent package and relationship state

filesystem and runtime
→ actual system result
```

## Contents

1. [Installing, Rebuilding, and Removing Modules](#1-installing-rebuilding-and-removing-modules)
2. [Inspecting Modules and System State](#2-inspecting-modules-and-system-state)
3. [Understanding Moonbase Modules](#3-understanding-moonbase-modules)
4. [Managing Dependencies and Optional Features](#4-managing-dependencies-and-optional-features)
5. [Configuration, Options, and Reconfiguration](#5-configuration-options-and-reconfiguration)
6. [Logs, Manifests, and Troubleshooting](#6-logs-manifests-and-troubleshooting)
7. [Caches, Archives, and Recovery](#7-caches-archives-and-recovery)
8. [Updating the System Safely](#8-updating-the-system-safely)
9. [Working with Moonbase](#9-working-with-moonbase)
10. [Advanced Module Inspection and Debugging](#10-advanced-module-inspection-and-debugging)
11. [Module Removal and Configuration Preservation](#11-module-removal-and-configuration-preservation)
12. [Plugins and Extensibility](#12-plugins-and-extensibility)
13. [Policy States: Held, Exiled, and Enforced Modules](#13-policy-states-held-exiled-and-enforced-modules)
14. [Building and Testing a Local Module](#14-building-and-testing-a-local-module)
15. [System Recovery and State Repair](#15-system-recovery-and-state-repair)
16. [Reference Commands and File Locations](#16-reference-commands-and-file-locations)

---

## 1. Installing, Rebuilding, and Removing Modules

### 1. Overview

LSS manages software through modules stored in Moonbase.

The three most common operations are:

```bash
lin module
lin -c module
lrm module
```

They mean:

```text
lin module
→ install a module

lin -c module
→ rebuild an installed module

lrm module
→ remove an installed module
```

These commands affect more than files on disk. They may also update:

```text
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/compile
/var/log/lunar/md5sum
/var/cache/lunar
/var/log/lunar/activity
```

Run them as root.

### 2. Before changing a module

Before installing, rebuilding, or removing a module, inspect its current state.

Useful commands:

```bash
lvu installed module
lvu where module
lvu depends module
lvu leert module
```

Example:

```bash
lvu installed xxhash
lvu where xxhash
lvu depends xxhash
lvu leert xxhash
```

These answer different questions:

```text
lvu installed
→ which version is installed?

lvu where
→ where is the module stored in Moonbase?

lvu depends
→ which installed modules depend on it?

lvu leert
→ what does its reverse dependency tree look like?
```

Do not remove a module only because it looks small or unfamiliar.

A module may be required indirectly by:

- the compiler toolchain;
- the boot process;
- networking;
- package management;
- another installed module;
- local system administration scripts.

### 3. Locating a module

Use:

```bash
lvu where module
```

Example:

```bash
lvu where foremost
```

Possible output:

```text
core/filesys
```

The full module directory is then typically:

```text
/var/lib/lunar/moonbase/core/filesys/foremost
```

Inspect its contents:

```bash
find /var/lib/lunar/moonbase/core/filesys/foremost   -maxdepth 2 -type f -print | sort
```

Common module files include:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

Not every module uses every lifecycle file.

### 4. Inspecting the module definition

Read `DETAILS` to see:

- version;
- source URL;
- checksums;
- description;
- maintainer metadata.

Read `BUILD` to understand:

- compiler flags;
- patches;
- configure steps;
- installation commands;
- files removed after installation;
- custom paths.

Example:

```bash
sed -n '1,240p'   /var/lib/lunar/moonbase/core/filesys/foremost/BUILD
```

For potentially sensitive modules, also inspect:

```bash
for hook in PRE_REMOVE POST_REMOVE PRE_INSTALL POST_INSTALL; do
  test -f "$MODULE_DIR/$hook" && {
    echo "=== $hook ==="
    cat "$MODULE_DIR/$hook"
  }
done
```

This is especially important before removing services, shells, boot components, or package-management tools.

### 5. Installing a module

Install with:

```bash
lin module
```

Example:

```bash
lin foremost
```

A normal installation may perform:

```text
resolve configuration
→ resolve dependencies
→ download sources
→ compile
→ remove an older installation if needed
→ install files
→ create ownership manifest
→ create checksum log
→ create compile log
→ create cache archive
→ update package state
```

Typical console phases include:

```text
Downloading source file ...
Building ...
Preparing to install ...
Installing ...
installation completed
Creating /var/log/lunar/compile/...
Creating /var/log/lunar/install/...
Creating /var/log/lunar/md5sum/...
Creating /var/cache/lunar/...
```

Not every module prints exactly the same messages.

### 6. Verifying an installation

After `lin` finishes, check the package-state record:

```bash
grep '^module:' /var/state/lunar/packages
```

Example:

```bash
grep '^foremost:' /var/state/lunar/packages
```

A record has this general form:

```text
module:date:state:version:size
```

Example:

```text
foremost:20260717:installed:1.5.7:116KB
```

Check the installed version:

```bash
lvu installed foremost
```

Inspect the ownership manifest:

```bash
cat /var/log/lunar/install/foremost-1.5.7
```

Check the installed command:

```bash
command -v foremost
```

For a library, use the appropriate file or linker inspection instead.

Examples:

```bash
ls -l /usr/lib/libxxhash.so*
pkgconf --modversion libxxhash
```

### 7. Understanding the install manifest

The install manifest is stored at:

```text
/var/log/lunar/install/module-version
```

It records the final paths attributed to the module.

It may contain:

- files;
- symbolic links;
- directories;
- documentation;
- manual pages;
- Lunar-generated logs.

It does not necessarily contain every path touched during the build.

A file can be:

```text
created during BUILD
→ observed by installwatch
→ removed before finalization
→ absent from final manifest
```

Therefore the manifest describes final ownership, not raw build history.

### 8. Installed size

The size stored in `/var/state/lunar/packages` is calculated from regular files listed in the final manifest.

It may include:

- executables;
- libraries;
- headers;
- documentation;
- manual pages;
- Lunar install log;
- Lunar compile log;
- Lunar MD5 log.

It does not directly count directories or symbolic links.

Therefore the recorded size should be understood as:

```text
regular filesystem content attributed to the module
```

not strictly:

```text
upstream application payload
```

A rebuild may recalculate this value even when the version and manifest remain unchanged.

### 9. Rebuilding an installed module

Use:

```bash
lin -c module
```

Example:

```bash
lin -c xxhash
```

A rebuild compiles and installs the module again.

For an already installed module, LSS uses an upgrade-style transition:

```text
build replacement
→ prepare installation
→ remove old ownership using upgrade semantics
→ install replacement
→ regenerate logs and manifest
→ update package state
```

Use rebuild when:

- testing changed compiler flags;
- validating a new toolchain;
- repairing a damaged installation;
- applying an updated module recipe;
- regenerating metadata;
- testing reproducibility.

### 10. Before a rebuild

Record the current state when the result matters.

Example:

```bash
MODULE=xxhash
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-check/$MODULE

mkdir -p "$BASE"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package.before"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.before"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/md5sum.before"
```

Then rebuild:

```bash
lin -c "$MODULE" 2>&1 | tee "$BASE/rebuild.log"
```

Compare afterward:

```bash
grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package.after"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.after"

diff -u "$BASE/install.before" "$BASE/install.after"
diff -u "$BASE/package.before" "$BASE/package.after"
```

No manifest difference means the final owned path set is unchanged.

The package record may still show a new date or recalculated size.

### 11. Rebuild safety

Before rebuilding a critical module, consider:

- available disk space;
- build time;
- source availability;
- whether the installed system depends on the module during the build;
- whether a cache exists;
- whether recovery media or a container snapshot is available.

Critical examples include:

```text
glibc
gcc
bash
coreutils
lunar
openssl
kernel
filesystem-related libraries
```

Prefer a disposable container or chroot for experiments.

### 12. Removing a module

Remove with:

```bash
lrm module
```

Example:

```bash
lrm foremost
```

Typical success output:

```text
Removed module: foremost
```

Normal removal is driven by the module's install manifest.

LSS uses that manifest to identify owned paths.

### 13. Before removing a module

Check reverse dependencies:

```bash
lvu depends module
lvu leert module
```

A safe standalone candidate may show no dependents.

Example:

```text
foremost:
```

This represents the root of the reverse tree with no dependent branches.

Also inspect dependency records:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Example:

```bash
grep -E '^foremost:|^[^:]+:foremost:'   /var/state/lunar/depends
```

No output means no matching stored relationships were found.

Do not treat that alone as proof that removal is safe. Also consider operational use outside LSS.

### 14. Preserve evidence before removal

Module-specific logs are normally removed with the module.

Copy them first when you need them:

```bash
MODULE=foremost
VERSION=$(lvu installed "$MODULE")
BACKUP=/root/lss-remove-backup/$MODULE

mkdir -p "$BACKUP"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BACKUP/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BACKUP/md5sum-log"

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BACKUP/" 2>/dev/null || true

grep "^${MODULE}:" /var/state/lunar/packages   > "$BACKUP/package-record"
```

This matters because `lrm` may delete:

```text
install manifest
compile log
MD5 log
```

### 15. What removal deletes

For a normal standalone module, LSS may remove:

- executable files;
- libraries;
- headers;
- documentation;
- manual pages;
- module-specific directories that become empty;
- module-specific Lunar logs;
- the module record from `/var/state/lunar/packages`;
- dependency records when applicable.

Validated example:

```text
/usr/bin/foremost
/usr/share/doc/foremost/CHANGES
/usr/share/doc/foremost/README
/usr/share/man/man8/foremost.8.gz
```

were removed.

### 16. What removal does not blindly delete

Shared directories remain when still required.

Validated examples:

```text
/usr
/usr/share
/usr/share/doc
```

remained after removing `foremost`.

An exclusive directory may disappear when empty:

```text
/usr/share/doc/foremost
```

Therefore the presence of a directory in the install manifest does not mean LSS will remove the entire directory tree regardless of other content.

### 17. Configuration files and `PRESERVE`

LSS treats files under `/etc` specially.

With:

```text
PRESERVE=on
```

the validated behavior is:

```text
configuration unchanged since installation
→ remove it with the module

configuration modified locally
→ preserve it
```

This protects local administrative changes without leaving every unmodified configuration behind.

### 18. Checking whether a configuration was modified

Use the module's MD5 log:

```text
/var/log/lunar/md5sum/module-version
```

For a simple manual check:

```bash
md5sum /etc/example.conf
grep '/etc/example.conf'   /var/log/lunar/md5sum/module-version
```

The exact format of the MD5 log should be inspected before scripting around it.

The decisive question is:

```text
does the current file match the checksum recorded at installation?
```

### 19. Preserved orphaned configuration

When a modified configuration survives `lrm`:

```text
the file remains
the package record disappears
the install and MD5 logs disappear
```

The file is now orphaned local state.

LSS no longer records an installed module as its owner.

Before reinstalling, decide whether to:

- keep it and let the new installation interact with it;
- back it up;
- compare it with the new default;
- remove it deliberately.

### 20. Reinstalling after removal

Use:

```bash
lin module
```

After reinstalling, verify:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

For a configuration file:

```bash
md5sum /etc/module.conf
```

When testing preservation behavior, remove or archive the orphaned modified file first if the goal is to restore the pristine default.

Example:

```bash
cp -a /etc/foremost.conf   /root/foremost.conf.saved

rm -f /etc/foremost.conf
lin foremost
```

### 21. Held, exiled, and enforced states

The package-state database supports more than `installed`.

Known state terms include:

```text
held
exiled
enforced
```

These are policy states and can affect normal operations.

Before changing a module, inspect its complete record:

```bash
grep '^module:' /var/state/lunar/packages
```

Do not assume that every installed-looking module is freely upgradeable or removable.

The exact operational handling of all policy-state combinations belongs in a separate chapter.

### 22. Caches

With archiving enabled, installation may create a cache such as:

```text
/var/cache/lunar/module-version-triplet.tar.xz
```

Check with:

```bash
ls -l /var/cache/lunar/module-*
```

A cache is not the same as an install manifest.

It is a reusable installation artifact.

The relevant objects are distinct:

```text
source archive
build tree
installation cache
install manifest
compile log
MD5 log
package-state record
```

### 23. Logs

Useful locations:

```text
/var/log/lunar/activity
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
```

Use the activity log for broader history:

```bash
grep 'module' /var/log/lunar/activity
```

Use the compile log for build failures:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Use the install manifest to inspect ownership:

```bash
cat /var/log/lunar/install/module-version
```

Use the MD5 log for `/etc` decisions and integrity comparison.

### 24. Troubleshooting a failed installation

When `lin` fails:

1. identify the reported phase;
2. inspect the compile log;
3. inspect the module's BUILD and lifecycle hooks;
4. verify compiler and linker selection;
5. verify source download and checksum;
6. verify available disk space;
7. check dependency state;
8. avoid repeated blind rebuilds.

Typical phase labels include:

```text
PRE_BUILD
BUILD
POST_BUILD
POST_INSTALL
```

Open the compile log:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

If the log was not finalized, inspect the path reported by `lin` or the temporary build area.

### 25. Troubleshooting an unexpected removal result

If a file remains after `lrm`, determine whether it is:

- a modified `/etc` file preserved by policy;
- a `PROTECTED` path;
- an `EXCLUDED` path that was never owned;
- a shared file or directory;
- recreated by another service;
- owned by another installed module;
- outside the old install manifest.

Check:

```bash
test -e /path && ls -ld /path
grep -Fx '/path' /root/saved-install-log
grep -R -F '/path' /var/log/lunar/install 2>/dev/null
```

The last command may identify another module manifest containing the same path.

### 26. A safe operational pattern

For an unfamiliar module:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
grep "^${MODULE}:" /var/state/lunar/packages
lvu depends "$MODULE"
lvu leert "$MODULE"
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

Preserve evidence:

```bash
mkdir -p "/root/module-backup/$MODULE"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "/root/module-backup/$MODULE/"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "/root/module-backup/$MODULE/"
```

Then perform only the intended operation:

```bash
lin "$MODULE"
```

or:

```bash
lin -c "$MODULE"
```

or:

```bash
lrm "$MODULE"
```

Verify immediately afterward.

### 27. Minimal verification checklist

### After install

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
test -f /var/log/lunar/install/module-version
```

### After rebuild

```bash
lvu installed module
diff old-install-log new-install-log
grep '^module:' /var/state/lunar/packages
```

### After removal

```bash
grep '^module:' /var/state/lunar/packages || true
test -e /known/payload/path && echo present || echo missing
```

### For configuration preservation

```bash
test -f /etc/module.conf && echo preserved || echo removed
```

### 28. Safety rules

1. Check reverse dependencies before removal.
2. Avoid testing on critical modules.
3. Preserve logs before `lrm`.
4. Use a container, chroot, snapshot, or backup for experiments.
5. Inspect module hooks.
6. Treat `/etc` files as administrative state.
7. Do not infer ownership only from filesystem presence.
8. Do not infer final ownership from raw install events.
9. Verify package and dependency databases after state-changing operations.
10. Restore instrumentation and test modifications when finished.

### 29. Summary

The normal operational model is:

```text
install
→ build and observe
→ record final ownership
→ register persistent state

rebuild
→ replace old owned state
→ regenerate ownership and metadata

remove
→ follow the ownership manifest
→ preserve modified configuration according to policy
→ remove package state and module artifacts
```

LSS combines flexible shell-based module recipes with filesystem observation and persistent ownership records.

For the user, the essential discipline is:

```text
inspect
→ preserve evidence
→ perform one operation
→ verify state
```

---

## 2. Inspecting Modules and System State

### 1. Overview

LSS keeps several complementary views of the system.

No single command or file tells the whole story.

The most useful inspection sources are:

```text
lvu
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/md5sum
/var/log/lunar/compile
/var/log/lunar/activity
/var/cache/lunar
Moonbase module files
```

Together they answer:

```text
what is installed?
which version?
which policy state?
what depends on it?
which files does it own?
which configuration files changed?
what happened during build?
what happened historically?
is a reusable cache available?
```

The correct operational habit is:

```text
inspect several views
→ compare them
→ interpret the result
```

### 2. `lvu` as the inspection interface

`lvu` is the main user-facing inspection tool.

It provides commands for:

- installed versions;
- Moonbase location;
- dependencies;
- reverse dependencies;
- module trees;
- module sections;
- sizes;
- module files;
- module history and metadata.

Useful starting commands:

```bash
lvu installed module
lvu where module
lvu depends module
lvu leert module
```

Example:

```bash
lvu installed xxhash
lvu where xxhash
lvu depends xxhash
lvu leert xxhash
```

These commands do not all read the same underlying state.

Their outputs should be interpreted according to the question being asked.

### 3. Checking whether a module is installed

Use:

```bash
lvu installed module
```

Example:

```bash
lvu installed foremost
```

Possible output:

```text
1.5.7
```

No useful output usually means the module is not currently installed.

For authoritative state, also inspect:

```bash
grep '^foremost:' /var/state/lunar/packages
```

Example record:

```text
foremost:20260717:installed:1.5.7:116KB
```

Use both when debugging inconsistent state.

### 4. Understanding `/var/state/lunar/packages`

The package-state database is:

```text
/var/state/lunar/packages
```

Observed record format:

```text
module:date:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

Fields:

```text
module
→ module name

date
→ state or installation date in YYYYMMDD form

state-list
→ installed and policy states

version
→ installed version

size
→ recorded installed size
```

Known state terms include:

```text
installed
held
exiled
enforced
```

A record may represent both current installation and policy.

Therefore this file is not merely an installed-package list.

### 5. Querying package state safely

Exact module lookup:

```bash
grep '^module:' /var/state/lunar/packages
```

Example:

```bash
grep '^xxhash:' /var/state/lunar/packages
```

List installed modules:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {print $1}'   /var/state/lunar/packages | sort
```

Show module, version, and size:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {
  printf "%-30s %-20s %s\n", $1, $4, $5
}' /var/state/lunar/packages
```

Show held modules:

```bash
awk -F: '$3 ~ /(^|\+)held(\+|$)/ {print}'   /var/state/lunar/packages
```

Do not edit this database manually during normal administration.

Use LSS commands so related state remains consistent.

### 6. Date semantics

The date field is refreshed during installation or rebuild.

Example:

```text
before rebuild:
xxhash:20260605:installed:0.8.3:31KB

after rebuild:
xxhash:20260717:installed:0.8.3:520KB
```

The version remained unchanged, but the date and size changed.

Therefore:

```text
same version
≠ unchanged package-state record
```

A rebuild can refresh metadata without changing the upstream version.

### 7. Size semantics

The recorded size is derived from regular files in the final install manifest.

It may include:

- executables;
- libraries;
- headers;
- documentation;
- manual pages;
- Lunar install log;
- Lunar compile log;
- Lunar MD5 log.

It does not directly count directories and symbolic links.

To reproduce the current calculation:

```bash
MODULE=xxhash
VERSION=$(lvu installed "$MODULE")
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "/var/log/lunar/install/${MODULE}-${VERSION}"

echo "${SIZE}KB"
```

Use `du -k` explicitly.

Interactive aliases such as:

```text
du='du -h'
```

can otherwise produce values like `100K`, which are not valid shell integers.

### 8. Locating a module in Moonbase

Use:

```bash
lvu where module
```

Example:

```bash
lvu where xxhash
```

Possible output:

```text
core/libs
```

Construct the full path:

```bash
MODULE=xxhash
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s\n' "$MODULE_DIR"
```

This path contains the module definition used by LSS.

### 9. Inspecting module files

List module files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

Common files:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

Read the main files:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

Inspect dependency declarations:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Inspect hooks:

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

### 10. Why module files matter

The persistent state tells you what LSS believes now.

The module files tell you what LSS intends to do next.

For example:

```text
packages
→ current installed state

depends
→ current dependency relationships

BUILD
→ future build and installation actions

PRE_REMOVE / POST_REMOVE
→ future removal side effects
```

A safe decision often requires both current state and module intent.

### 11. Understanding `/var/state/lunar/depends`

The dependency-state database is:

```text
/var/state/lunar/depends
```

Observed format:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

Interpretation:

```text
module
→ ccache

dependency
→ xxhash

status
→ on

type
→ required

field5 and field6
→ additional declaration parameters
```

The final two fields may be empty.

Always preserve all six fields when analyzing or exporting records.

### 12. Inspecting direct dependency records

Show dependencies declared by a module:

```bash
grep '^module:' /var/state/lunar/depends
```

Example:

```bash
grep '^ccache:' /var/state/lunar/depends
```

Show modules that depend on a target:

```bash
grep -E '^[^:]+:target:' /var/state/lunar/depends
```

Example:

```bash
grep -E '^[^:]+:xxhash:' /var/state/lunar/depends
```

Show both directions:

```bash
MODULE=xxhash

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

This is useful before rebuild or removal.

### 13. `lvu depends`

Use:

```bash
lvu depends module
```

This shows installed modules that depend on the target.

Example:

```bash
lvu depends xxhash
```

Observed output included:

```text
ccache
```

This is a direct operational warning:

```text
removing xxhash
→ may affect ccache
```

Depending on the system, other modules may also appear.

### 14. `lvu leert`

Use:

```bash
lvu leert module
```

This shows the reverse dependency tree.

Example structure:

```text
xxhash: ccache dav1d kitty python-xxhash ...
^----ccache: ...
```

The tree helps identify indirect impact.

A leaf candidate may show only:

```text
foremost:
```

That means the tree has a root but no dependent branches.

Do not treat a dependency tree as the only safety test.

External scripts and operational use may not appear in LSS dependency state.

### 15. Leaf modules

Leaf modules are installed modules without recorded reverse dependents.

List them:

```bash
lvu leafs | sort
```

Leaf status is useful for:

- cleanup review;
- low-risk experiments;
- identifying optional software;
- removal candidates.

Leaf does not mean:

```text
unimportant
safe under every circumstance
unused outside LSS
```

Examples of technically leaf modules may still include:

- boot tools;
- network tools;
- package-management tools;
- administrative commands;
- shared libraries used outside declared Moonbase relationships.

Inspect role and module files before removal.

### 16. The install manifest

Per-module ownership manifests are stored under:

```text
/var/log/lunar/install
```

File name:

```text
module-version
```

Example:

```text
/var/log/lunar/install/foremost-1.5.7
```

Inspect:

```bash
cat /var/log/lunar/install/foremost-1.5.7
```

Count entries:

```bash
wc -l /var/log/lunar/install/foremost-1.5.7
```

The manifest describes final paths attributed to the module.

### 17. Classifying manifest entries

Use:

```bash
MANIFEST=/var/log/lunar/install/foremost-1.5.7

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link\t%s\t%s\n' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file\t%s\n' "$path"
  elif [ -d "$path" ]; then
    printf 'directory\t%s\n' "$path"
  else
    printf 'missing\t%s\n' "$path"
  fi
done < "$MANIFEST"
```

This distinguishes:

```text
regular files
symbolic links
directories
missing paths
```

A missing manifest path can indicate:

- manual deletion;
- later replacement;
- incomplete state;
- module behavior after installation;
- filesystem damage.

### 18. Ownership versus raw build activity

The install manifest is not a raw event log.

Installwatch may observe:

```text
create
chmod
rename
unlink
```

for a path during BUILD.

If the path no longer exists when final ownership is created, it may not appear in the manifest.

Validated example:

```text
/usr/lib/libxxhash.a
```

was installed and later unlinked during the same build.

It appeared in raw installwatch evidence but not in the final manifest.

Therefore:

```text
raw build activity
≠ final ownership
```

### 19. Finding possible shared ownership

Search all manifests for a path:

```bash
grep -R -F -x '/usr/bin/example'   /var/log/lunar/install 2>/dev/null
```

Example:

```bash
grep -R -F -x '/usr/lib/libexample.so'   /var/log/lunar/install 2>/dev/null
```

Multiple matches indicate potential overlapping ownership.

Overlapping ownership requires careful interpretation:

- one module may replace another;
- a path may be intentionally shared;
- state may be stale;
- manifests may reflect different installation eras.

Do not manually delete such paths without understanding the overlap.

### 20. MD5 logs

Per-module checksum logs are stored under:

```text
/var/log/lunar/md5sum
```

Example:

```text
/var/log/lunar/md5sum/foremost-1.5.7
```

Inspect:

```bash
cat /var/log/lunar/md5sum/foremost-1.5.7
```

These logs help LSS decide whether configuration files under `/etc` were locally modified.

They are also useful for:

- integrity checks;
- troubleshooting;
- comparing current state with installed state.

### 21. Checking a configuration file

Current checksum:

```bash
md5sum /etc/foremost.conf
```

Search the module's MD5 record:

```bash
grep '/etc/foremost.conf'   /var/log/lunar/md5sum/foremost-1.5.7
```

Interpretation:

```text
current checksum matches installed checksum
→ unchanged

current checksum differs
→ locally modified or otherwise changed
```

With `PRESERVE=on`, a modified `/etc` file is preserved during normal removal.

### 22. Compile logs

Compile logs are stored under:

```text
/var/log/lunar/compile
```

Typical file:

```text
module-version.xz
```

Read with:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search errors:

```bash
xzgrep -n -i   -E 'error|failed|undefined|not found'   /var/log/lunar/compile/module-version.xz
```

Compile logs may contain:

- compiler commands;
- configure output;
- linker errors;
- patch failures;
- install output;
- lifecycle stage messages.

They are often the first source to inspect after a failed `lin`.

### 23. Activity log

The broader system activity log is:

```text
/var/log/lunar/activity
```

Search by module:

```bash
grep 'foremost' /var/log/lunar/activity
```

Follow live activity:

```bash
tail -f /var/log/lunar/activity
```

The activity log provides historical transitions such as:

- installation;
- rebuild;
- removal;
- failure;
- version changes.

Unlike module-specific logs, it normally remains after the module is removed.

### 24. Cache inspection

Caches are stored under:

```text
/var/cache/lunar
```

Find a module cache:

```bash
ls -l /var/cache/lunar/module-*
```

Example:

```bash
ls -l /var/cache/lunar/xxhash-*
```

A cache file may include:

```text
module
version
target triplet
compression suffix
```

The cache is distinct from the install manifest.

The manifest describes ownership.

The cache contains reusable installation content.

### 25. Comparing state before and after an operation

Create a small evidence directory:

```bash
MODULE=foremost
BASE=/root/lss-state-check/$MODULE

mkdir -p "$BASE/before" "$BASE/after"
```

Before:

```bash
cp -a /var/state/lunar/packages "$BASE/before/packages"
cp -a /var/state/lunar/depends "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true
```

After:

```bash
cp -a /var/state/lunar/packages "$BASE/after/packages"
cp -a /var/state/lunar/depends "$BASE/after/depends"
```

Compare:

```bash
diff -u "$BASE/before/packages" "$BASE/after/packages"
diff -u "$BASE/before/depends" "$BASE/after/depends"
```

This reveals exact persistent-state transitions.

### 26. Comparing manifests

Before rebuild:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.before"
```

After rebuild:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.after"
```

Compare:

```bash
diff -u "$BASE/install.before" "$BASE/install.after"
```

No output means the manifests are byte-for-byte identical.

This does not necessarily mean package metadata is unchanged.

### 27. Inspecting filesystem state from a manifest

Capture:

```bash
while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link\t%s\t%s\n' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file\t%s\n' "$path"
  elif [ -d "$path" ]; then
    printf 'directory\t%s\n' "$path"
  else
    printf 'missing\t%s\n' "$path"
  fi
done < "$MANIFEST" > filesystem-state.txt
```

Repeat after an operation and compare:

```bash
diff -u filesystem-state.before filesystem-state.after
```

This is especially useful for removal testing.

### 28. Detecting stale metadata

Possible signs:

```text
recorded size differs greatly from current calculation
manifest contains missing files
installed version differs from module metadata
package record exists but manifest is missing
manifest exists but package record is absent
dependency record references a missing module
```

Example checks:

```bash
MODULE=xxhash
VERSION=$(lvu installed "$MODULE")

test -f "/var/log/lunar/install/${MODULE}-${VERSION}" ||
  echo "missing install manifest"

grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "missing package record"
```

Do not repair persistent state by hand before preserving evidence.

First determine how the inconsistency arose.

### 29. Cross-checking installed and Moonbase versions

Installed version:

```bash
lvu installed module
```

Moonbase version:

```bash
lvu version module
```

Example:

```bash
printf 'installed: %s\n' "$(lvu installed xxhash)"
printf 'moonbase:  %s\n' "$(lvu version xxhash)"
```

Possible interpretations:

```text
same
→ current according to active Moonbase

different
→ upgrade available, local divergence, or stale Moonbase
```

Use the active Moonbase as the reference, not an unrelated repository copy.

### 30. Inspecting sections

Show a module's section:

```bash
lvu where module
```

List a section:

```bash
lvu section section-name
```

Use `lvu where` for a module name.

Do not use:

```bash
lvu section module-name
```

unless the module name is also a valid section name.

A section error does not imply the module is missing.

### 31. Inspecting multiple modules

For a compact installed-state report:

```bash
for module in xxhash foremost ccache; do
  printf '\n=== %s ===\n' "$module"
  printf 'installed: %s\n' "$(lvu installed "$module" 2>/dev/null)"
  printf 'section:   %s\n' "$(lvu where "$module" 2>/dev/null)"
  grep "^${module}:" /var/state/lunar/packages || true
  grep -E "^${module}:|^[^:]+:${module}:"     /var/state/lunar/depends || true
done
```

This is useful before a toolchain or dependency transition.

### 32. A module inspection template

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== INSTALLED VERSION ==='
printf '%s\n' "$VERSION"

echo
echo '=== MOONBASE LOCATION ==='
printf '%s\n' "$MODULE_DIR"

echo
echo '=== REVERSE DEPENDENTS ==='
lvu depends "$MODULE"
lvu leert "$MODULE"

echo
echo '=== DEPENDENCY RECORDS ==='
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends || true

echo
echo '=== MODULE FILES ==='
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort

echo
echo '=== INSTALL MANIFEST ==='
if [ -n "$VERSION" ] &&
   [ -f "/var/log/lunar/install/${MODULE}-${VERSION}" ]; then
  cat "/var/log/lunar/install/${MODULE}-${VERSION}"
fi
```

This template provides a reliable first-pass inspection.

### 33. Interpreting absence

No output can mean different things.

Examples:

```text
lvu depends module
→ no reverse dependents

grep packages
→ no current package record

grep depends
→ no matching dependency records

missing manifest
→ uninstalled, stale state, or damaged logs

missing module directory
→ inactive Moonbase, removed module, or wrong section
```

Always identify which source produced no output.

Do not collapse all absence into “not installed.”

### 34. Source of truth by question

Use the right source for the right question.

```text
Is the module recorded as installed?
→ /var/state/lunar/packages

Which version is installed?
→ lvu installed + packages record

Where is the module definition?
→ lvu where

What does the module declare?
→ Moonbase files

What currently depends on it?
→ lvu depends, lvu leert, depends database

Which paths does it own?
→ install manifest

Was an /etc file modified?
→ current checksum + MD5 log

Why did compilation fail?
→ compile log

What happened historically?
→ activity log

Can installation be resurrected?
→ cache directory
```

### 35. Common mistakes

### Mistake 1: treating `packages` as only an installed list

It also contains policy state.

### Mistake 2: treating the install manifest as raw build history

It describes final ownership.

### Mistake 3: using `du` on manifest directories

This recursively counts unrelated filesystem content.

### Mistake 4: trusting leaf status alone

Operational dependencies may exist outside LSS records.

### Mistake 5: removing before copying logs

`lrm` may delete the evidence.

### Mistake 6: assuming no command output means failure

Some inspection commands legitimately return nothing.

### Mistake 7: editing state files manually

This may desynchronize LSS state.

### 36. Minimal daily inspection set

For routine administration:

```bash
lvu installed module
lvu where module
lvu depends module
grep '^module:' /var/state/lunar/packages
cat /var/log/lunar/install/module-version
```

For risky changes, add:

```bash
lvu leert module
grep dependency records
inspect BUILD and hooks
copy install and MD5 logs
snapshot packages and depends
```

### 37. Summary

LSS exposes system truth through several coordinated artifacts.

The core inspection model is:

```text
Moonbase
→ intended module behavior

packages
→ current installation and policy state

depends
→ current relationship state

install manifest
→ final owned paths

MD5 log
→ installed checksums

compile log
→ build evidence

activity log
→ historical transitions

cache
→ reusable installation artifact
```

Reliable administration comes from comparing these views rather than trusting one in isolation.

The practical rule is:

```text
ask one precise question
→ choose the correct state source
→ cross-check when the change is risky

---

## 3. Understanding Moonbase Modules

### 1. Overview

Moonbase is the collection of module definitions used by LSS.

A module describes how software is:

- identified;
- downloaded;
- configured;
- built;
- installed;
- integrated;
- removed;
- related to other modules.

A module is not only metadata and not only a shell script.

It is a lifecycle definition.

Typical module directory:

```text
/var/lib/lunar/moonbase/<section>/<module>
```

Example:

```text
/var/lib/lunar/moonbase/core/filesys/foremost
```

### 2. Locating a module

Use:

```bash
lvu where module
```

Example:

```bash
lvu where foremost
```

Output:

```text
core/filesys
```

Construct the full path:

```bash
MODULE=foremost
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s\n' "$MODULE_DIR"
```

List its files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

### 3. Common module files

A module may contain:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
CONFLICTS
patches
auxiliary files
plugin files
```

Not every file is required.

The most common minimum is:

```text
DETAILS
BUILD
```

The presence of extra files means the module extends the default lifecycle.

### 4. `DETAILS`

`DETAILS` defines module identity and source metadata.

It commonly contains fields such as:

```text
MODULE
VERSION
SOURCE
SOURCE_URL
SOURCE_VFY
WEB_SITE
ENTERED
UPDATED
SHORT
cat << EOF
long description
EOF
```

Exact fields may vary by module age and conventions.

Typical responsibilities:

```text
module name
version
source archive name
download location
integrity verification
upstream website
short description
long description
maintenance timestamps
```

Inspect with:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
```

### 5. Why `DETAILS` matters

`DETAILS` answers:

```text
what is this module?
which version does Moonbase define?
where does the source come from?
how is the source verified?
what does the software do?
```

Before upgrading or debugging a download failure, inspect:

- `VERSION`;
- `SOURCE`;
- `SOURCE_URL`;
- verification fields;
- upstream project location.

A mismatch between installed and Moonbase versions is normal when an update is available.

### 6. `BUILD`

`BUILD` defines build and installation actions.

Example pattern:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

Many modules use shared helpers:

```bash
default_build
default_make
prepare_install
sedit
```

Inspect with:

```bash
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

### 7. The critical role of `prepare_install`

`prepare_install` marks the semantic boundary between:

```text
build-local activity
```

and:

```text
system installation activity
```

Typical structure:

```bash
configure
make
prepare_install
make install
```

Before `prepare_install`, files are normally created inside the build tree.

After `prepare_install`, installwatch observes filesystem operations performed against the system.

A custom `BUILD` that installs files before calling `prepare_install` may bypass normal ownership tracking.

### 8. Default build helpers

Common helpers reduce repeated shell code.

### `default_build`

Usually performs a conventional sequence equivalent to:

```text
configure
→ make
→ prepare_install
→ make install
```

The exact implementation belongs to the loaded LSS function libraries.

### `default_make`

Usually handles projects that need only:

```text
make
→ prepare_install
→ make install
```

A module using a default helper is still worth inspecting because it may alter variables or patch files before the helper call.

### 9. Example: `foremost`

Observed `BUILD`:

```bash
export CFLAGS+=" -fcommon"
sedit 's:/usr/local:/usr:g;s:/usr/man:/usr/share/man:g;s:/usr/etc:/etc:g' Makefile config.c &&
sedit "s:-O2:$CFLAGS:" Makefile &&
default_make
```

Interpretation:

```text
add compiler compatibility flag
→ normalize installation paths
→ inject Lunar compiler flags
→ run default make/install lifecycle
```

This module had no custom remove hooks.

### 10. Example: `xxhash`

The reference module was useful because its build installed a static archive and then removed it.

Conceptually:

```text
make
→ prepare_install
→ make install
→ remove /usr/lib/libxxhash.a
```

Installwatch observed the file lifecycle, but the final manifest omitted the removed archive.

This shows why reading the complete `BUILD` matters.

The final state may differ from the immediate result of `make install`.

### 11. `DEPENDS`

`DEPENDS` declares module relationships.

It may express:

- required dependencies;
- optional dependencies;
- enabled or disabled selections;
- purpose-specific parameters;
- conditional choices.

Inspect with:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

The persistent result is represented in:

```text
/var/state/lunar/depends
```

But the source declaration and the persistent state are not the same thing.

### 12. Declaration versus persistent dependency state

`DEPENDS` describes what the module can or should depend on.

`/var/state/lunar/depends` describes the resolved stored relationship for the current system.

Therefore:

```text
DEPENDS
→ declaration and choices

depends database
→ current resolved state
```

A change in `DEPENDS` does not automatically prove that the persistent database has already changed.

The module must be configured, installed, or otherwise processed.

### 13. Required and optional dependencies

A required dependency is necessary for the selected module configuration.

An optional dependency may enable or disable functionality.

Before removing a module, inspect both:

```bash
lvu depends target
lvu leert target
```

Also inspect direct records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

A module may be operationally important even when it is only an optional dependency.

### 14. `CONFIGURE`

`CONFIGURE` is used when a module needs interactive or stored configuration decisions.

It may ask questions such as:

- enable a feature;
- select a backend;
- choose an implementation;
- include optional integration;
- use a particular build mode.

The answers can influence:

```text
dependencies
compiler flags
configure switches
installed files
runtime features
```

Inspect with:

```bash
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"
```

### 15. `OPTIONS`

`OPTIONS` may define selectable build options in a structured way.

Depending on module conventions, options may map user choices to:

```text
configure flags
feature toggles
dependency activation
environment variables
```

Inspect with:

```bash
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

Configuration and options should be reviewed before comparing two builds.

Two systems can build the same module version with materially different results.

### 16. `PRE_BUILD`

`PRE_BUILD` runs before the main `BUILD`.

Typical uses:

- patch source files;
- generate files;
- adjust environment;
- remove bundled components;
- prepare build directories;
- apply compatibility fixes.

Inspect with:

```bash
test -f "$MODULE_DIR/PRE_BUILD" &&
  sed -n '1,240p' "$MODULE_DIR/PRE_BUILD"
```

A failure here is reported as a `PRE_BUILD` lifecycle failure.

### 17. `POST_BUILD`

`POST_BUILD` runs after `BUILD` and before final installation processing completes.

Typical uses:

- remove unwanted installed files;
- adjust generated files;
- create symlinks;
- normalize permissions;
- perform cleanup that must affect final ownership.

This stage is particularly important because it can change the final manifest.

A file installed during `BUILD` and removed during `POST_BUILD` may appear in raw installwatch activity but not in the final ownership manifest.

### 18. `POST_INSTALL`

`POST_INSTALL` runs after the core installation transaction.

The module lock is released before this stage.

Typical uses:

- print notices;
- refresh external indexes;
- run integration commands;
- perform actions that should happen after ownership state is committed.

A failure here is distinct from a build failure.

The software may already be installed even if `POST_INSTALL` reports a problem.

### 19. `PRE_REMOVE`

`PRE_REMOVE` runs before normal ownership removal.

Typical uses:

- stop a service;
- unregister integration;
- prepare data migration;
- remove generated runtime state;
- warn the administrator.

Inspect before removing sensitive modules:

```bash
test -f "$MODULE_DIR/PRE_REMOVE" &&
  sed -n '1,240p' "$MODULE_DIR/PRE_REMOVE"
```

A remove operation can have effects not visible in the install manifest when hooks are present.

### 20. `POST_REMOVE`

`POST_REMOVE` runs after the main removal operation.

Typical uses:

- refresh caches;
- rebuild indexes;
- remove residual integration;
- print administrative guidance.

Inspect with:

```bash
test -f "$MODULE_DIR/POST_REMOVE" &&
  sed -n '1,240p' "$MODULE_DIR/POST_REMOVE"
```

Do not assume that `lrm` only deletes manifest paths.

Hooks can add lifecycle-specific actions.

### 21. `CONFLICTS`

`CONFLICTS` declares modules or conditions that cannot coexist safely.

Possible reasons:

- duplicate implementations;
- overlapping files;
- incompatible daemons;
- competing providers;
- mutually exclusive toolchains.

Inspect with:

```bash
test -f "$MODULE_DIR/CONFLICTS" &&
  sed -n '1,240p' "$MODULE_DIR/CONFLICTS"
```

Conflict handling happens before the main installation transaction.

### 22. Patches and auxiliary files

A module may ship patches or helper files in its directory.

List everything:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

Common auxiliary content:

```text
*.patch
*.diff
configuration templates
service files
desktop files
scripts
udev rules
plugin files
```

The presence of a file does not prove it is used.

Search references:

```bash
grep -R -n -F 'filename' "$MODULE_DIR"
```

### 23. Installed plugins

A Moonbase module may install LSS plugin files under a plugin directory such as:

```text
plugin.d
```

This means a module can extend LSS itself.

Before removing such a module, inspect:

- installed plugin files;
- module manifest;
- plugin registration;
- current running `lin` processes.

Plugins may be reloaded through a signal path involving `USR1`.

A plugin-bearing module deserves more caution than a simple leaf utility.

### 24. Shell semantics

Moonbase modules are executable shell logic.

This gives them flexibility, but also means behavior can depend on:

- environment variables;
- shell functions;
- command availability;
- path layout;
- toolchain selection;
- configuration state;
- previously installed files.

Read modules as programs, not static metadata.

Important shell features include:

```text
&& chains
conditional branches
variable expansion
command substitution
function calls
environment export
file mutation
```

### 25. Understanding `&&` chains

Example:

```bash
step_one &&
step_two &&
step_three
```

Meaning:

```text
run step_one
→ only if successful, run step_two
→ only if successful, run step_three
```

A failure stops the chain.

When diagnosing a module, identify the first failed command rather than only the final lifecycle label.

### 26. Environment variables

Modules commonly modify:

```text
CFLAGS
CXXFLAGS
LDFLAGS
PATH
CC
CXX
MAKEFLAGS
```

Example:

```bash
export CFLAGS+=" -fcommon"
```

Toolchain-sensitive modules may require explicit:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

A toolchain mismatch can produce errors unrelated to the upstream source.

Always inspect module-local exports and global Lunar configuration.

### 27. `sedit`

`sedit` is a Lunar helper used to edit files in place.

Example:

```bash
sedit 's:/usr/local:/usr:g' Makefile
```

Operationally:

```text
modify source/build file before compilation or installation
```

When a generated path looks unexpected, search the module for `sedit` calls.

### 28. Module path conventions

Modules are grouped into sections.

Examples:

```text
core/libs
core/filesys
core/utils
```

Section names organize Moonbase.

The module name remains the normal command argument:

```bash
lin foremost
```

not:

```bash
lin core/filesys/foremost
```

Use `lvu where` to map name to location.

### 29. Reading a module before installation

Minimal review:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== DETAILS ==='
sed -n '1,240p' "$MODULE_DIR/DETAILS"

echo
echo '=== BUILD ==='
sed -n '1,240p' "$MODULE_DIR/BUILD"

echo
echo '=== DEPENDS ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"

echo
echo '=== HOOKS ==='
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  echo "--- $hook"
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

### 30. Reading a module before removal

Focus on:

```text
reverse dependencies
PRE_REMOVE
POST_REMOVE
installed manifest
configuration files
services and plugins
shared files
```

Template:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

lvu depends "$MODULE"
lvu leert "$MODULE"

for hook in PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" &&
    sed -n '1,240p' "$MODULE_DIR/$hook"
done

cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

### 31. Reading a module during troubleshooting

When build fails:

1. note the lifecycle phase;
2. inspect compile log;
3. inspect `BUILD`;
4. inspect `PRE_BUILD`;
5. inspect toolchain variables;
6. inspect patches;
7. inspect source version and checksum;
8. compare with a known working module pattern.

Useful searches:

```bash
grep -R -n -E 'CC=|CXX=|CFLAGS|LDFLAGS|prepare_install|default_'   "$MODULE_DIR"
```

### 32. Module behavior versus final ownership

A module can:

- create temporary files;
- remove installed files;
- generate files dynamically;
- install shared paths;
- modify external state;
- run commands after installation.

The final install manifest only captures attributed final paths.

It does not fully describe:

- every command executed;
- every transient file;
- every external side effect;
- every service action;
- every plugin action.

For complete understanding, combine:

```text
module source
compile log
raw installwatch when instrumented
final manifest
persistent state
activity log
```

### 33. Safe assumptions

Usually safe:

```text
DETAILS defines module identity and source
BUILD defines main build/install logic
DEPENDS defines dependency declarations
hooks extend lifecycle stages
install manifest records final owned paths
```

Not safe without inspection:

```text
module has no side effects
leaf means harmless
same version means same build
no manifest difference means no metadata change
no dependency records means no operational dependency
```

### 34. A compact anatomy map

```text
DETAILS
→ identity, version, source, description

CONFIGURE / OPTIONS
→ user choices and build variation

DEPENDS
→ dependency declarations

PRE_BUILD
→ prepare source and environment

BUILD
→ compile and install

POST_BUILD
→ adjust final installed state

POST_INSTALL
→ actions after core installation

PRE_REMOVE
→ actions before ownership removal

POST_REMOVE
→ actions after removal

CONFLICTS
→ incompatible modules or states

patches / auxiliary files
→ implementation support
```

### 35. User checklist

Before install:

```text
read DETAILS
read BUILD
inspect DEPENDS
inspect options
note toolchain requirements
```

Before rebuild:

```text
record installed version
save old manifest
check changed module files
check compiler environment
```

Before removal:

```text
check reverse dependencies
read remove hooks
save logs
inspect /etc ownership
check services and plugins
```

### 36. Summary

A Moonbase module is an executable lifecycle definition.

Its structure connects:

```text
source metadata
→ configuration
→ dependency decisions
→ build logic
→ observed installation
→ final ownership
→ removal behavior
```

The practical rule is:

```text
do not judge a module by its name or size
→ read the module
→ inspect current state
→ understand its lifecycle role

---

## 4. Managing Dependencies and Optional Features

### 1. Overview

Dependency handling is one of the defining strengths of LSS.

LSS does not reduce every dependency to a fixed package list.

A module may express:

- required dependencies;
- optional dependencies;
- feature-dependent dependencies;
- alternative providers;
- enabled or disabled relationships;
- stored user choices;
- additional parameters associated with a dependency.

This allows the user to decide how much functionality a module should include.

The same module version can therefore produce different builds on different systems.

### 2. Why optional dependencies matter

Many package managers install a predefined dependency closure.

LSS allows the dependency graph to reflect the user's intended system.

Conceptually:

```text
module
→ required core
→ optional capabilities
→ user selection
→ resolved dependency graph
→ resulting build
```

This supports systems that are:

- smaller;
- more purpose-specific;
- easier to audit;
- less burdened by unused integrations;
- closer to the administrator's actual intent.

Optional dependencies are not merely a convenience.

They are part of system design.

### 3. Required dependencies

A required dependency is necessary for the selected module configuration.

Conceptually:

```text
module A
requires module B
→ B must be available before A can be built or used correctly
```

A stored dependency record may look like:

```text
ccache:xxhash:on:required::
```

Interpretation:

```text
module
→ ccache

dependency
→ xxhash

status
→ on

type
→ required
```

Required dependencies normally participate in ordering and installation planning.

### 4. Optional dependencies

An optional dependency enables additional functionality but is not necessary for the module's minimum usable form.

Examples of optional capabilities may include:

- graphical interfaces;
- database backends;
- compression formats;
- image formats;
- audio support;
- network protocols;
- scripting language bindings;
- documentation tools;
- hardware integration;
- authentication mechanisms.

Conceptually:

```text
optional dependency enabled
→ dependency is installed or activated
→ matching build feature is enabled

optional dependency disabled
→ dependency is omitted
→ matching build feature is disabled
```

The exact mechanism is defined by the module.

### 5. Optional does not mean unimportant

An optional dependency may still be operationally critical for a particular user.

Example:

```text
TLS support may be optional in the module definition
but essential for the administrator's intended use
```

Therefore:

```text
optional in Moonbase
≠ irrelevant in deployment
```

The administrator should evaluate functionality, not only dependency type.

### 6. Dependency declarations

Dependency declarations are commonly stored in:

```text
DEPENDS
```

Locate the module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

A module without a `DEPENDS` file may still rely on:

- bootstrap components;
- shell tools;
- implicit toolchain requirements;
- runtime commands not represented as formal dependencies.

### 7. Declaration versus resolved state

The module's `DEPENDS` file describes available or required relationships.

The current system's resolved dependency state is stored in:

```text
/var/state/lunar/depends
```

This distinction is fundamental:

```text
DEPENDS
→ what the module declares or permits

depends database
→ what the current system resolved and stored
```

Do not assume that reading `DEPENDS` alone reveals the active build choices.

### 8. Persistent dependency record format

Observed records use six colon-separated fields:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

The first four fields are operationally clear:

```text
module
dependency
status
type
```

The final fields preserve additional declaration parameters and may be empty.

When parsing, preserve all fields.

Example:

```bash
awk -F: '{
  printf "module=%s dependency=%s status=%s type=%s extra1=%s extra2=%s\n",
         $1, $2, $3, $4, $5, $6
}' /var/state/lunar/depends
```

### 9. Enabled and disabled relationships

The status field may represent whether a dependency selection is active.

Conceptually:

```text
on
→ selected and active

off
→ known choice, currently disabled
```

A disabled optional dependency may remain represented because the user's choice itself is persistent state.

This is important:

```text
absence of a relationship
≠ explicitly disabled relationship
```

An explicit negative choice can be meaningful during reconfiguration.

### 10. Inspecting a module's current dependencies

Show records where the module is the depender:

```bash
MODULE=name

grep "^${MODULE}:" /var/state/lunar/depends
```

Show modules that depend on it:

```bash
grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends
```

Show both directions:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

This provides the raw stored relationships.

### 11. Using `lvu`

Useful commands:

```bash
lvu depends module
lvu leert module
```

`lvu depends` shows installed modules that depend on the target.

`lvu leert` displays the reverse dependency tree.

Example:

```bash
lvu depends xxhash
lvu leert xxhash
```

These commands help estimate the impact of rebuild or removal.

### 12. Forward and reverse views

Two questions must be kept separate:

```text
What does this module depend on?
→ forward dependency view

What depends on this module?
→ reverse dependency view
```

Forward view matters before installation and rebuild.

Reverse view matters before removal or replacement.

A module can have few forward dependencies but many reverse dependents.

### 13. Dependency ordering

LSS constructs dependency relationships and uses graph ordering for multi-module operations.

Conceptually:

```text
dependency graph
→ topological ordering
→ build dependencies before dependents
```

`tsort` is used in the dependency-ordering machinery.

A valid dependency graph must not contain unresolved cycles that prevent ordering.

### 14. Multi-module installation

For several requested modules, LSS separates work into stages:

```text
configure and resolve dependencies
→ download sources
→ build and install sequentially
```

This allows downloads to overlap while keeping installation state changes ordered.

Dependencies may enlarge the actual set of modules beyond the names entered by the user.

### 15. User choices and build results

Optional dependency choices may affect:

- configure arguments;
- compiler definitions;
- linked libraries;
- installed files;
- runtime commands;
- plugin availability;
- service integration;
- package size;
- future reverse dependencies.

Therefore:

```text
same module name
+ same version
+ different optional choices
→ potentially different software
```

This is expected behavior, not inconsistency.

### 16. Inspecting configuration files

Dependency choices may be implemented through:

```text
DEPENDS
CONFIGURE
OPTIONS
BUILD
```

Inspect all relevant files:

```bash
for file in DEPENDS CONFIGURE OPTIONS BUILD; do
  if [ -f "$MODULE_DIR/$file" ]; then
    echo "=== $file ==="
    sed -n '1,240p' "$MODULE_DIR/$file"
  fi
done
```

Search for feature flags:

```bash
grep -R -n -E   'optional_depends|required_depends|depends|enable|disable|with-|without-'   "$MODULE_DIR"
```

Exact helper names and conventions may vary.

### 17. Optional dependency pattern

A typical optional dependency conceptually maps:

```text
dependency selected
→ add dependency record
→ install dependency if needed
→ pass --enable-feature or --with-library

dependency rejected
→ store disabled state when applicable
→ pass --disable-feature or --without-library
```

The module may also use environment variables or custom shell branches.

Read the module rather than assuming a single universal syntax.

### 18. Reconfiguration

Changing optional features usually requires the module to be configured again and rebuilt.

Conceptually:

```text
old choices
→ reconfigure
→ dependency graph changes
→ rebuild
→ new manifest and package state
```

A dependency choice does not alter an already compiled binary merely by editing a state file.

The module must pass through its build lifecycle again.

### 19. Before changing optional features

Preserve current state:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-option-change/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/before/forward-depends" || true

grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends   > "$BASE/before/reverse-depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true
```

Also record the module files:

```bash
cp -a "$MODULE_DIR" "$BASE/before/module-definition"
```

### 20. After reconfiguration

Capture:

```bash
grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/after/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/after/forward-depends" || true

grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends   > "$BASE/after/reverse-depends" || true

VERSION_AFTER=$(lvu installed "$MODULE")

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

Compare:

```bash
diff -u   "$BASE/before/forward-depends"   "$BASE/after/forward-depends"

diff -u   "$BASE/before/install-log"   "$BASE/after/install-log"

diff -u   "$BASE/before/package-record"   "$BASE/after/package-record"
```

### 21. What to expect after changing an option

Possible changes:

```text
new dependency record
removed dependency record
on → off
off → on
new linked library
new executable or plugin
removed executable or plugin
different package size
different install manifest
same version but different build
```

A version number alone cannot describe the full configuration.

### 22. Removing an optional dependency

Do not immediately run:

```bash
lrm optional-library
```

First determine whether dependent modules were built with it enabled.

Inspect:

```bash
lvu depends optional-library
lvu leert optional-library
```

Then inspect the dependent modules' stored records:

```bash
grep -E "^[^:]+:optional-library:"   /var/state/lunar/depends
```

If the relationship is active, first reconfigure or rebuild the dependent module without that feature.

Safe conceptual order:

```text
disable feature in dependent module
→ rebuild dependent module
→ verify relationship is removed or off
→ remove optional dependency
```

### 23. Orphaned dependencies

A dependency may remain installed after no module depends on it.

This can happen after:

- disabling an optional feature;
- removing a dependent module;
- changing providers;
- rebuilding with fewer features.

Use:

```bash
lvu leafs
```

to identify modules without reverse dependents.

But leaf status does not automatically mean unnecessary.

The module may be directly useful to the administrator.

### 24. Optional dependency cleanup

A careful cleanup process:

```text
identify leaf
→ inspect package role
→ inspect Moonbase definition
→ search external usage
→ preserve logs
→ remove only after review
```

Useful checks:

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -R -n -w "$MODULE"   /etc /usr/local /root 2>/dev/null | head -100
```

The search is only a clue, not definitive proof of use.

### 25. Alternative providers

Some capabilities may be satisfied by alternative modules.

Conceptually:

```text
feature requires capability X
→ provider A or provider B
```

The selected provider affects:

- dependency state;
- installed files;
- runtime behavior;
- conflict handling;
- future rebuilds.

Before switching providers:

1. inspect conflicts;
2. inspect reverse dependents;
3. preserve current state;
4. configure the replacement;
5. rebuild affected modules;
6. verify runtime behavior;
7. remove the old provider only after validation.

### 26. Dependency conflicts

Conflicts may be declared separately from dependencies.

Inspect:

```bash
test -f "$MODULE_DIR/CONFLICTS" &&
  sed -n '1,240p' "$MODULE_DIR/CONFLICTS"
```

A dependency choice may activate a conflict indirectly.

Do not force both providers into the filesystem unless the module design explicitly supports coexistence.

### 27. Hidden or implicit dependencies

Not all operational requirements are necessarily recorded as formal dependency relationships.

Possible implicit requirements:

- shell commands assumed by BUILD;
- compiler toolchain;
- kernel features;
- hardware;
- service manager;
- external configuration;
- local scripts;
- manually installed software.

This is why dependency state is necessary but not sufficient for risk analysis.

### 28. Build-time versus runtime dependencies

A dependency may be needed only to build the module.

Another may be required at runtime.

Some may be needed for both.

This distinction matters for cleanup.

Removing a build-only dependency after installation may be safe in some cases, but only if:

- no future rebuild is expected without reinstalling it;
- no runtime linkage exists;
- no other module needs it;
- the stored policy allows it.

Do not infer dependency phase only from package size or name.

Inspect the declaration and build commands.

### 29. Checking runtime linkage

For an executable:

```bash
ldd /usr/bin/program
```

For a shared library:

```bash
readelf -d /usr/lib/libexample.so | grep NEEDED
```

For pkg-config metadata:

```bash
pkgconf --print-requires package
pkgconf --print-requires-private package
```

These commands provide runtime or link metadata, not LSS policy state.

Compare them with `/var/state/lunar/depends`.

### 30. Dependency state and truth

The dependency database records LSS's resolved relationship state.

It does not prove:

- that the dependency is physically present;
- that linking succeeded;
- that runtime use is functional;
- that no undeclared dependency exists;
- that the declaration is still current.

Cross-check when accuracy matters:

```text
depends database
→ intended/resolved relationship

packages database
→ installed module state

filesystem/linker inspection
→ physical implementation

runtime test
→ actual functionality
```

### 31. Detecting inconsistencies

Potential problems:

```text
active dependency record but dependency not installed
installed dependency but no reverse relationships
dependent binary linked against missing library
disabled record but build still contains feature
Moonbase declaration changed but stored choice is old
```

Checks:

```bash
DEP=xxhash

grep -E "^[^:]+:${DEP}:on:" /var/state/lunar/depends

grep "^${DEP}:" /var/state/lunar/packages ||
  echo "dependency package record missing"
```

For a program:

```bash
ldd /usr/bin/program | grep 'not found'
```

Preserve evidence before repairing state.

### 32. Optional features and reproducibility

To reproduce a build, recording only:

```text
module name
version
```

is insufficient.

Also preserve:

- dependency selections;
- configuration answers;
- options;
- compiler flags;
- provider choices;
- active Moonbase revision;
- toolchain versions.

A more complete build identity is:

```text
module
+ version
+ Moonbase definition
+ selected dependencies
+ options
+ toolchain
+ environment
```

### 33. A dependency review template

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== DECLARATION ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"

echo
echo '=== STORED FORWARD RELATIONSHIPS ==='
grep "^${MODULE}:" /var/state/lunar/depends || true

echo
echo '=== REVERSE RELATIONSHIPS ==='
grep -E "^[^:]+:${MODULE}:"   /var/state/lunar/depends || true

echo
echo '=== REVERSE DEPENDENCY VIEW ==='
lvu depends "$MODULE"
lvu leert "$MODULE"

echo
echo '=== OPTIONS AND CONFIGURATION ==='
for file in CONFIGURE OPTIONS; do
  test -f "$MODULE_DIR/$file" || continue
  echo "--- $file"
  sed -n '1,240p' "$MODULE_DIR/$file"
done
```

### 34. Safe optional-feature change pattern

```text
1. Inspect declaration.
2. Inspect stored dependency state.
3. Record current manifest and package state.
4. Change the option through LSS.
5. Rebuild the module.
6. Compare dependency records.
7. Compare final manifest.
8. Test the affected feature.
9. Remove newly unused dependencies only after review.
```

This avoids breaking a working module by removing a dependency too early.

### 35. Common mistakes

### Mistake 1: installing every optional dependency

This defeats one of LSS's main design advantages.

### Mistake 2: disabling features only to reduce package count

A small dependency may provide essential functionality.

### Mistake 3: removing a dependency before rebuilding the dependent module

The existing binary may still require it.

### Mistake 4: treating `DEPENDS` as the current system state

It is a declaration, not the complete resolved state.

### Mistake 5: treating `/var/state/lunar/depends` as physical proof

It is persistent LSS state, not a runtime test.

### Mistake 6: comparing builds only by version

Optional choices can change the result substantially.

### Mistake 7: assuming leaf means orphan

A leaf may be directly installed for user use.

### 36. Design value of optional dependencies

Optional dependencies let Lunar systems remain intentionally different.

A workstation may enable:

```text
graphical interfaces
audio
printing
media formats
desktop integration
```

A server may omit them and enable:

```text
TLS
database support
network services
monitoring
```

A rescue environment may choose only:

```text
filesystem tools
network diagnostics
recovery utilities
```

The module remains the same logical module, but its resolved form reflects the machine's purpose.

### 37. Summary

LSS dependency management is not only package ordering.

It is a configuration mechanism.

The operational model is:

```text
module declaration
→ required and optional relationships
→ user choice
→ stored dependency state
→ ordered build
→ feature-specific installed result
```

The central rule is:

```text
enable what the system needs
→ omit what it does not need
→ preserve the choices
→ rebuild before removing dependencies
```

Optional dependencies are one of the mechanisms through which Lunar Linux remains both source-based and administrator-directed.

---

## 5. Configuration, Options, and Reconfiguration

### 1. Overview

LSS allows a module to be configured according to the purpose of the system.

Configuration may control:

- optional features;
- dependency choices;
- providers;
- compiler flags;
- build modes;
- installed components;
- integration with other software;
- runtime capabilities.

The same module version may therefore produce different results on different Lunar systems.

This is expected and intentional.

### 2. Configuration as part of package identity

For a binary-only package manager, package identity is often reduced to:

```text
name
version
architecture
```

For LSS, a more complete identity is:

```text
module
+ version
+ Moonbase definition
+ selected dependencies
+ selected options
+ compiler and linker flags
+ toolchain
+ environment
```

Two installations of the same version can differ materially if their configuration differs.

### 3. Configuration sources

Module configuration may be expressed through:

```text
CONFIGURE
OPTIONS
DEPENDS
BUILD
global Lunar configuration
local Lunar configuration
environment variables
plugin decisions
```

Not every module uses all these mechanisms.

A simple module may only contain `DETAILS` and `BUILD`.

A complex module may combine several layers.

### 4. `CONFIGURE`

`CONFIGURE` contains module-specific configuration logic.

It may ask the user to:

- enable or disable functionality;
- select an implementation;
- choose an optional backend;
- enable language bindings;
- choose a service integration;
- select a build variant.

Inspect it with:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"
```

Read the full logic, not only the visible question text.

The answer may trigger other state changes.

### 5. `OPTIONS`

`OPTIONS` defines build options in a reusable or structured form.

Options may map a user choice to:

```text
--enable-feature
--disable-feature
--with-library
--without-library
compiler definitions
environment variables
conditional build branches
```

Inspect:

```bash
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

The exact syntax depends on LSS conventions and module age.

### 6. `DEPENDS` as configuration

Optional dependencies are also configuration choices.

A dependency selection may determine whether a feature is built.

Conceptually:

```text
enable feature
→ enable dependency
→ add build switch
→ build extra functionality
```

or:

```text
disable feature
→ disable dependency
→ add negative build switch
→ omit functionality
```

Therefore `DEPENDS` is not only a graph declaration.

It can also be part of the module's configuration interface.

### 7. `BUILD` as final interpretation

Configuration decisions are ultimately consumed by `BUILD` and its helpers.

Typical patterns include:

```bash
OPTS+=" --enable-feature"
```

or:

```bash
if module_installed dependency; then
  OPTS+=" --with-dependency"
else
  OPTS+=" --without-dependency"
fi
```

The stored answer alone does not change installed software.

The module must be rebuilt so the build logic can apply the choice.

### 8. Global configuration

Global LSS behavior is controlled through files such as:

```text
/etc/lunar/config
```

Observed settings include:

```text
ARCHIVE
PRESERVE
REAP
```

Other global settings may affect:

- compiler flags;
- architecture;
- optimization;
- source locations;
- build directories;
- logging;
- cache behavior;
- color and sound;
- parallelism.

Inspect:

```bash
sed -n '1,260p' /etc/lunar/config
```

Do not change global settings casually before rebuilding a large part of the system.

### 9. Local configuration

LSS may load local configuration after its main configuration and function libraries.

Local configuration can override defaults or add site-specific behavior.

This is useful for:

- system-wide policy;
- local mirrors;
- build flags;
- organization-specific paths;
- hardware-specific choices.

Document local changes.

Otherwise later rebuild differences may be difficult to explain.

### 10. Environment variables

The build environment may influence a module even when no explicit option is selected.

Common variables:

```text
CC
CXX
CFLAGS
CXXFLAGS
LDFLAGS
MAKEFLAGS
PATH
PKG_CONFIG_PATH
```

Example:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

A toolchain mismatch can produce failures that appear to be module bugs.

Before a sensitive rebuild, record:

```bash
env | sort > build-environment.txt
```

For a more focused view:

```bash
env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS|PATH|PKG_CONFIG_PATH)='
```

### 11. Persistent choices

LSS preserves dependency and option choices so they can be reused.

This provides continuity across:

- rebuilds;
- upgrades;
- repeated installations;
- system-wide updates.

Persistent choice means:

```text
the system remembers administrator intent
```

It does not mean the choice can never be changed.

### 12. Explicit disabled choices

An optional relationship may be stored as disabled rather than omitted.

Conceptually:

```text
dependency absent from state
→ no recorded decision

dependency stored as off
→ explicit decision not to use it
```

This distinction matters during reconfiguration and automation.

A stored negative choice prevents the system from repeatedly asking or silently changing intent.

### 13. Inspecting active configuration

There is no single universal file that describes every module's complete active configuration.

Use several sources:

```text
module CONFIGURE
module OPTIONS
module DEPENDS
stored dependency records
package record
install manifest
compile log
global configuration
environment
```

Template:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== DEPENDENCY STATE ==='
grep "^${MODULE}:" /var/state/lunar/depends || true

echo
echo '=== CONFIGURE ==='
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"

echo
echo '=== OPTIONS ==='
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"

echo
echo '=== DEPENDS ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

### 14. When reconfiguration is needed

Reconfigure when:

- enabling a previously disabled feature;
- disabling an active feature;
- changing an optional provider;
- changing a build backend;
- changing compiler or linker policy;
- changing a major global build setting;
- testing a different module variant;
- adapting the module to new hardware or services.

A simple source update may not require new answers if the intended configuration remains unchanged.

### 15. Reconfiguration requires rebuild

Changing configuration state does not rewrite existing binaries.

The correct lifecycle is:

```text
change choice
→ resolve dependency changes
→ rebuild module
→ create new manifest
→ update package state
→ test functionality
```

If dependent libraries are removed before the rebuild, the current installed binary may break.

### 16. Safe reconfiguration workflow

### Step 1: record current state

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-reconfigure/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/before/depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

env | sort > "$BASE/before/environment"
```

### Step 2: inspect the module

```bash
for file in CONFIGURE OPTIONS DEPENDS BUILD; do
  test -f "$MODULE_DIR/$file" || continue
  cp -a "$MODULE_DIR/$file" "$BASE/before/"
done
```

### Step 3: change the choice through LSS

Use the normal LSS configuration path.

Do not edit `/var/state/lunar/depends` manually.

### Step 4: rebuild

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild.log"
```

### Step 5: capture new state

```bash
VERSION_AFTER=$(lvu installed "$MODULE")

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/after/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/after/depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

### Step 6: compare

```bash
diff -u "$BASE/before/depends" "$BASE/after/depends"
diff -u "$BASE/before/install-log" "$BASE/after/install-log"
diff -u "$BASE/before/package-record" "$BASE/after/package-record"
```

### Step 7: test the feature

Do not stop at a successful build.

Test the capability that was enabled or disabled.

### 17. Enabling a feature

Safe sequence:

```text
inspect module
→ enable option
→ install new dependency if required
→ rebuild module
→ inspect new manifest
→ verify runtime feature
```

Possible results:

- new libraries linked;
- new executable installed;
- new plugin installed;
- new dependency records;
- larger package size;
- different configuration files.

### 18. Disabling a feature

Safe sequence:

```text
disable option
→ rebuild dependent module
→ verify feature is absent
→ verify dependency relationship changed
→ remove dependency only if no longer needed
```

Do not remove the dependency first.

The old binary may still link against it.

### 19. Switching providers

A provider change can affect more than one module.

Safe order:

```text
inspect current provider
→ inspect conflicts
→ enable replacement provider
→ rebuild affected modules
→ test runtime behavior
→ disable old provider relationships
→ remove old provider after verification
```

A provider may be selected through:

- optional dependency;
- configuration question;
- build option;
- conflict rule;
- plugin.

### 20. Configuration and manifests

A changed configuration may change the final ownership manifest.

Examples:

```text
enable documentation
→ new documentation paths

enable language bindings
→ new library and module paths

enable service integration
→ new service files

disable static libraries
→ fewer library files
```

Compare manifests before and after.

The manifest is the clearest evidence of final filesystem differences.

### 21. Configuration and package size

Package size may change after reconfiguration.

This may reflect:

- added files;
- removed files;
- different binary sizes;
- additional documentation;
- new plugins;
- new Lunar artifacts.

The version may remain the same.

Therefore:

```text
same version
+ new options
→ new package-state size
```

### 22. Configuration and dependencies

A configuration choice may produce:

```text
new required dependency
new optional dependency
on → off
off → on
provider replacement
removed dependency record
```

Inspect the raw state:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
```

Also inspect reverse impact:

```bash
lvu depends dependency
lvu leert dependency
```

### 23. Configuration and reproducibility

To reproduce a configured module, preserve:

- module name;
- module version;
- active Moonbase revision;
- `DETAILS`;
- `BUILD`;
- `DEPENDS`;
- `CONFIGURE`;
- `OPTIONS`;
- dependency-state records;
- global configuration;
- local configuration;
- relevant environment variables;
- toolchain versions.

A minimal reproducibility record:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
OUT="/root/reproduce-$MODULE"

mkdir -p "$OUT"

cp -a "$MODULE_DIR" "$OUT/module"
grep "^${MODULE}:" /var/state/lunar/packages   > "$OUT/package-record"
grep "^${MODULE}:" /var/state/lunar/depends   > "$OUT/dependency-records"
cp -a /etc/lunar/config "$OUT/lunar-config"
env | sort > "$OUT/environment"
```

### 24. Reconfiguration and caches

An existing installation cache may reflect old configuration choices.

A cache built with one feature set should not automatically be assumed equivalent to a newly requested configuration.

Before using or trusting a cache, consider:

```text
module version
architecture
build options
dependency selection
toolchain
environment
```

Cache identity may not encode every meaningful build decision.

When configuration changes materially, a fresh build is safer.

### 25. Reconfiguration and `/etc`

A rebuild may interact with existing configuration files.

Possible outcomes depend on:

- installation behavior;
- MD5 state;
- module logic;
- preservation policy;
- upstream install rules.

Before rebuilding a service or daemon:

```bash
cp -a /etc/module.conf /root/module.conf.before
```

Compare afterward:

```bash
diff -u /root/module.conf.before /etc/module.conf
```

Do not assume a rebuild leaves local configuration untouched.

### 26. Reconfiguration and services

A newly enabled feature may require:

- service restart;
- service enablement;
- socket activation;
- new user or group;
- changed permissions;
- regenerated cache or index.

Inspect `POST_INSTALL`, service files, and module documentation.

LSS installation success does not prove the running service is using the new feature.

### 27. Reconfiguration and plugins

A configuration change may install or remove plugins.

This can affect:

- LSS itself;
- desktop environments;
- interpreters;
- applications;
- service frameworks.

When plugin paths change:

1. compare manifests;
2. identify host application;
3. reload or restart when required;
4. verify plugin discovery;
5. inspect runtime logs.

### 28. Detecting active build features

Possible methods:

```text
program --version
program --help
pkgconf metadata
linked libraries
installed plugins
configuration report
runtime feature test
```

Examples:

```bash
program --version
program --help | grep -i feature
ldd /usr/bin/program
pkgconf --print-requires package
```

Use the method appropriate to the software.

### 29. Troubleshooting ignored choices

If a selected option appears to have no effect:

1. inspect `CONFIGURE` and `OPTIONS`;
2. inspect stored dependency state;
3. inspect compile log for configure switches;
4. verify the module was rebuilt;
5. verify a cache did not restore the old build;
6. inspect the final manifest;
7. test runtime behavior;
8. inspect whether the upstream build system auto-detected a different result.

Search compile log:

```bash
xzgrep -n -E   'enable|disable|with-|without-|checking for|found|not found'   /var/log/lunar/compile/module-version.xz
```

### 30. Troubleshooting unexpected auto-detection

Some upstream build systems enable features automatically when a library is present.

This can override the administrator's expectation if the module does not pass an explicit negative flag.

Example pattern:

```text
library installed
→ configure auto-detects it
→ feature enabled
```

Even if no active LSS dependency record was expected.

Inspect:

- configure output;
- `BUILD`;
- `OPTIONS`;
- linked libraries;
- final manifest.

Explicit `--without-feature` or `--disable-feature` is more reproducible than relying on absence.

### 31. Troubleshooting stale choices

Possible symptoms:

- module keeps using an old provider;
- disabled dependency returns after rebuild;
- configuration question is not asked;
- package manifest does not match current intent.

Preserve state, then inspect:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
grep -R -n -F "$MODULE" /etc/lunar /var/state/lunar
```

Do not delete state records blindly.

First determine which LSS command is responsible for resetting or reconfiguring the module.

### 32. Configuration drift

Configuration drift occurs when:

```text
stored intent
≠ current module declaration
≠ installed build
≠ runtime behavior
```

Causes may include:

- Moonbase update;
- manual file changes;
- interrupted rebuild;
- provider replacement;
- stale cache;
- toolchain change;
- undeclared auto-detection.

A drift review should compare all four layers.

### 33. Configuration layers

A useful model:

```text
Layer 1: administrator intent
→ selected features and providers

Layer 2: persistent LSS choices
→ dependency and option state

Layer 3: build interpretation
→ flags and commands used

Layer 4: installed ownership
→ manifest and package record

Layer 5: runtime reality
→ actual program behavior
```

Reliable administration verifies that these layers agree.

### 34. Safe global configuration changes

A change to global compiler or optimization settings may affect many modules.

Before changing:

1. record current `/etc/lunar/config`;
2. record toolchain versions;
3. select a small representative module;
4. rebuild in a container or chroot;
5. inspect logs and runtime;
6. expand gradually;
7. avoid full-system rebuild until the setting is validated.

This reduces the blast radius.

### 35. Common mistakes

### Mistake 1: editing persistent state files manually

This can desynchronize choices and module behavior.

### Mistake 2: changing an option without rebuilding

The installed software remains unchanged.

### Mistake 3: removing an optional dependency before rebuilding

The current binary may still need it.

### Mistake 4: using package version as complete configuration identity

It omits important build choices.

### Mistake 5: trusting a cache after a major configuration change

The cache may represent old choices.

### Mistake 6: assuming build success proves feature success

Runtime validation is still necessary.

### Mistake 7: changing global flags and rebuilding everything immediately

Validate on a small module first.

### 36. Reconfiguration checklist

Before:

```text
inspect CONFIGURE, OPTIONS, DEPENDS, BUILD
record package and dependency state
save manifest and configuration
record environment and toolchain
```

During:

```text
change choice through LSS
allow dependencies to resolve
watch configure and build output
```

After:

```text
compare dependency state
compare manifest
compare package record
verify configuration files
test runtime feature
remove unused dependency only after review
```

### 37. Summary

Configuration in LSS is an operational contract between the administrator and the module.

The lifecycle is:

```text
administrator choice
→ persistent option and dependency state
→ build interpretation
→ final installed result
→ runtime verification
```

The central rule is:

```text
change intent through LSS
→ rebuild
→ compare state
→ test reality
```

This preserves one of Lunar Linux's strongest qualities: software is built for the actual purpose of the system rather than for a universal predefined feature set.

---

## 6. Logs, Manifests, and Troubleshooting

### 1. Overview

LSS preserves several kinds of operational evidence.

The most important are:

```text
compile log
install manifest
MD5 log
activity log
package-state database
dependency-state database
installation cache
console output
```

Each answers a different question.

A reliable troubleshooting process starts by identifying the exact question before choosing the evidence source.

### 2. The main evidence locations

```text
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
/var/log/lunar/activity
/var/state/lunar/packages
/var/state/lunar/depends
/var/cache/lunar
```

Typical per-module files:

```text
/var/log/lunar/compile/module-version.xz
/var/log/lunar/install/module-version
/var/log/lunar/md5sum/module-version
```

These files are related, but not interchangeable.

### 3. Compile log

The compile log records build-time output.

It may contain:

- configure output;
- compiler commands;
- warnings;
- errors;
- linker commands;
- install output;
- patch failures;
- lifecycle-stage messages;
- toolchain selection.

Read it with:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search for common failures:

```bash
xzgrep -n -i   -E 'error|failed|undefined|not found|cannot|fatal'   /var/log/lunar/compile/module-version.xz
```

Do not stop at the final error line.

Look for the first meaningful failure.

### 4. Lifecycle error boundaries

LSS associates failures with lifecycle stages such as:

```text
PRE_BUILD
BUILD
POST_BUILD
POST_INSTALL
```

These labels narrow the search.

### `PRE_BUILD`

Likely causes:

- failed patch;
- missing helper;
- bad source-tree assumption;
- environment preparation failure.

### `BUILD`

Likely causes:

- compiler error;
- configure failure;
- linker error;
- make failure;
- installation command failure.

### `POST_BUILD`

Likely causes:

- missing installed file;
- failed cleanup;
- failed symlink creation;
- permission adjustment failure.

### `POST_INSTALL`

Likely causes:

- cache/index refresh;
- service integration;
- registration step;
- post-transaction command.

A `POST_INSTALL` failure may occur after the core payload is already installed.

### 5. Console output

Console output is the first visible summary of the operation.

Preserve it when the operation matters:

```bash
lin module 2>&1 | tee install-console.log
```

For rebuild:

```bash
lin -c module 2>&1 | tee rebuild-console.log
```

For removal:

```bash
lrm module 2>&1 | tee remove-console.log
```

Console output is useful for chronology.

It is usually less complete than the compile log.

### 6. Install manifest

The install manifest records final owned paths:

```text
/var/log/lunar/install/module-version
```

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

Count entries:

```bash
wc -l /var/log/lunar/install/module-version
```

Use it to answer:

```text
which paths does LSS attribute to this module?
```

Do not use it to answer:

```text
which files were ever touched during BUILD?
```

### 7. Raw events versus final ownership

Installwatch observes filesystem operations during installation.

The final manifest is created after interpretation and filtering.

A path may be:

```text
created
→ modified
→ deleted
```

during the same build.

Such a path may appear in raw installwatch evidence but not in the final manifest.

Validated example:

```text
/usr/lib/libxxhash.a
```

was installed and later unlinked.

It was absent from the final manifest.

### 8. Manifest classification

Classify each entry:

```bash
MANIFEST=/var/log/lunar/install/module-version

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link\t%s\t%s\n' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file\t%s\n' "$path"
  elif [ -d "$path" ]; then
    printf 'directory\t%s\n' "$path"
  else
    printf 'missing\t%s\n' "$path"
  fi
done < "$MANIFEST"
```

This helps detect stale or damaged ownership state.

### 9. Missing manifest paths

A missing path may indicate:

- manual deletion;
- later module replacement;
- filesystem damage;
- stale manifest;
- a post-install action;
- another package taking ownership;
- incomplete cleanup.

Do not immediately recreate or delete state.

First determine whether the path is:

- expected;
- shared;
- replaced;
- protected;
- excluded;
- recreated elsewhere.

### 10. Comparing manifests

Before rebuild:

```bash
cp -a /var/log/lunar/install/module-version   install.before
```

After rebuild:

```bash
cp -a /var/log/lunar/install/module-version   install.after
```

Compare:

```bash
diff -u install.before install.after
```

No output means byte-for-byte identity.

A changed manifest may show:

- new feature;
- removed feature;
- different documentation;
- new plugin;
- changed path layout;
- packaging regression.

### 11. MD5 log

The MD5 log stores installed checksums:

```text
/var/log/lunar/md5sum/module-version
```

Inspect:

```bash
cat /var/log/lunar/md5sum/module-version
```

It is especially important for `/etc`.

LSS uses it to distinguish:

```text
unchanged configuration
modified configuration
```

### 12. Verifying a configuration file

Current checksum:

```bash
md5sum /etc/example.conf
```

Stored checksum:

```bash
grep '/etc/example.conf'   /var/log/lunar/md5sum/module-version
```

Interpretation:

```text
same checksum
→ unchanged

different checksum
→ locally modified or otherwise altered
```

With `PRESERVE=on`, a modified configuration is preserved during normal removal.

### 13. Activity log

The activity log is:

```text
/var/log/lunar/activity
```

Search by module:

```bash
grep 'module' /var/log/lunar/activity
```

Follow live:

```bash
tail -f /var/log/lunar/activity
```

It provides broader historical context:

- install;
- rebuild;
- removal;
- failure;
- version transition;
- lifecycle result.

Unlike module-specific logs, it usually survives `lrm`.

### 14. Package-state database

The package-state database is:

```text
/var/state/lunar/packages
```

Inspect one module:

```bash
grep '^module:' /var/state/lunar/packages
```

Record format:

```text
module:date:state-list:version:size
```

Use it to answer:

```text
is the module recorded as installed?
which version?
which policy state?
what size is recorded?
```

It is not a raw filesystem inventory.

### 15. Dependency-state database

The dependency-state database is:

```text
/var/state/lunar/depends
```

Inspect both directions:

```bash
MODULE=name

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Use it to answer:

```text
which relationships are stored?
which are active?
which type?
```

It does not prove runtime functionality.

### 16. Cache evidence

Caches are stored under:

```text
/var/cache/lunar
```

Inspect:

```bash
ls -l /var/cache/lunar/module-*
```

A cache can explain why a module was restored without a normal compile path.

When troubleshooting unexpected results, determine whether LSS:

```text
compiled from source
or
resurrected from cache
```

### 17. A complete evidence capture

Before a risky operation:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-evidence/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

After the operation, capture the same state again.

### 18. Diagnosing a failed build

A practical sequence:

```text
1. Read console output.
2. Identify lifecycle phase.
3. Open compile log.
4. Find first real error.
5. Inspect BUILD and hooks.
6. Inspect toolchain variables.
7. Inspect dependencies.
8. Verify source and checksum.
9. Reproduce only after forming a hypothesis.
```

Avoid blind repeated rebuilds.

They can overwrite useful evidence.

### 19. Compiler errors

Typical signs:

```text
undeclared identifier
unknown option
unsupported flag
missing header
type mismatch
syntax error
```

Check:

```bash
echo "$CC"
echo "$CXX"
echo "$CFLAGS"
echo "$CXXFLAGS"
```

Search module:

```bash
grep -R -n -E 'CC=|CXX=|CFLAGS|CXXFLAGS'   "$MODULE_DIR"
```

A GCC/Clang mismatch can produce misleading failures.

### 20. Linker errors

Typical signs:

```text
undefined reference
cannot find -l...
DSO missing from command line
symbol not found
```

Check:

```bash
echo "$LDFLAGS"
ldd /path/to/binary
readelf -d /path/to/binary | grep NEEDED
```

Inspect dependency state and pkg-config metadata.

### 21. Missing dependency

Typical signs:

```text
header not found
library not found
pkg-config package missing
command not found
```

Check:

```bash
grep '^module:' /var/state/lunar/depends
grep '^dependency:' /var/state/lunar/packages
command -v required-command
pkgconf --exists package
```

A stored dependency record may exist even when the dependency is physically missing.

### 22. Configure failure

Search:

```bash
xzgrep -n -E   'checking|not found|cannot|failed|error'   /var/log/lunar/compile/module-version.xz
```

Inspect build flags.

A feature may be auto-detected differently after a dependency or toolchain change.

### 23. Patch failure

Typical signs:

```text
Hunk FAILED
Reversed patch
file not found
malformed patch
```

Inspect:

- `PRE_BUILD`;
- patch files;
- source version;
- source-tree layout.

A patch may be stale after a version bump.

### 24. Installation failure

Typical signs:

```text
permission denied
no such file or directory
destination missing
install target failed
```

Check whether `prepare_install` was called before installation.

Inspect:

- path rewrites;
- DESTDIR use;
- permissions;
- filesystem availability;
- installwatch environment.

### 25. Post-build failure

A POST_BUILD failure may occur after files were installed.

Inspect:

- final manifest presence;
- raw console chronology;
- installed files;
- POST_BUILD script;
- package-state record.

Do not assume the system is unchanged.

### 26. Post-install failure

A POST_INSTALL failure may leave the module installed but incompletely integrated.

Check:

```bash
grep '^module:' /var/state/lunar/packages
cat /var/log/lunar/install/module-version
```

Then inspect:

- service state;
- cache refresh;
- registration step;
- plugin discovery;
- printed notices.

### 27. Diagnosing unexpected removal

If a file remains:

```text
modified /etc file
PROTECTED path
EXCLUDED path
shared ownership
recreated by service
not present in old manifest
```

Check:

```bash
grep -Fx '/path' saved-install-log
grep -R -F -x '/path' /var/log/lunar/install
```

If the file disappeared unexpectedly, inspect:

- old manifest;
- protection policy;
- remove hooks;
- shared ownership;
- preserved evidence.

### 28. Diagnosing preserved configuration

If `/etc/file` remains after `lrm`:

```bash
md5sum /etc/file
grep '/etc/file' saved-md5sum-log
```

A checksum difference with `PRESERVE=on` explains preservation.

The file is then local orphaned state.

### 29. Diagnosing stale package state

Possible symptoms:

```text
package record exists, manifest missing
manifest exists, package record missing
recorded version differs from file names
recorded size is implausible
```

Capture evidence first.

Then compare:

```bash
grep '^module:' /var/state/lunar/packages
ls -l /var/log/lunar/install/module-*
ls -l /var/log/lunar/md5sum/module-*
ls -l /var/log/lunar/compile/module-*
```

Do not hand-edit databases before understanding the inconsistency.

### 30. Diagnosing stale dependency state

Possible symptoms:

```text
dependency record references missing module
reverse tree disagrees with actual installed state
disabled choice appears active
old provider remains selected
```

Check:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends

grep '^dependency:' /var/state/lunar/packages
```

Inspect current `DEPENDS`, `CONFIGURE`, and `OPTIONS`.

### 31. Reproducing installed size

Use only regular files:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "/var/log/lunar/install/${MODULE}-${VERSION}"

echo "${SIZE}KB"
```

Do not run recursive `du` over manifest directories.

That counts unrelated content under `/usr`, `/usr/lib`, and other shared trees.

### 32. Raw installwatch capture

The raw installwatch file is temporary and normally destroyed.

Capturing it requires controlled instrumentation.

This is an advanced debugging technique.

Use it only in:

- a container;
- a chroot;
- a disposable test system;
- a backed-up environment.

The raw stream can reveal:

- transient files;
- unlink operations;
- renames;
- paths filtered from final ownership.

Restore the original LSS code after the experiment.

### 33. Comparing before and after state

Capture:

```bash
cp -a /var/state/lunar/packages packages.before
cp -a /var/state/lunar/depends depends.before
```

After operation:

```bash
cp -a /var/state/lunar/packages packages.after
cp -a /var/state/lunar/depends depends.after
```

Compare:

```bash
diff -u packages.before packages.after
diff -u depends.before depends.after
```

This provides exact persistent-state transitions.

### 34. Troubleshooting by question

### Did compilation fail?

```text
compile log
console output
BUILD and PRE_BUILD
toolchain environment
```

### Did installation ownership differ?

```text
install manifest
raw installwatch if captured
POST_BUILD
EXCLUDED and PROTECTED
```

### Did package state change?

```text
/var/state/lunar/packages
activity log
```

### Did dependency state change?

```text
/var/state/lunar/depends
DEPENDS
CONFIGURE
OPTIONS
```

### Did configuration survive removal?

```text
MD5 log
current checksum
PRESERVE setting
```

### Was cache used?

```text
console output
cache directory
activity history
```

### 35. Common diagnostic mistakes

### Mistake 1: reading only the last error line

The first meaningful failure is usually more useful.

### Mistake 2: treating console output as complete evidence

The compile log may contain more detail.

### Mistake 3: treating manifest as raw history

It records final ownership.

### Mistake 4: running recursive `du` on directory entries

This produces misleading totals.

### Mistake 5: removing a module before saving logs

`lrm` may destroy them.

### Mistake 6: editing state databases immediately

This destroys evidence and may worsen inconsistency.

### Mistake 7: rebuilding repeatedly without changing the hypothesis

This adds noise rather than knowledge.

### 36. A disciplined troubleshooting loop

```text
observe
→ preserve evidence
→ identify lifecycle phase
→ form one hypothesis
→ make one controlled change
→ rerun
→ compare
→ accept or reject hypothesis
```

This is faster and safer than random experimentation.

### 37. Minimal troubleshooting bundle

For support or review, preserve:

```text
console output
compile log
install manifest
MD5 log
package record
dependency records
module DETAILS
module BUILD
module hooks
relevant configuration
toolchain environment
```

Archive:

```bash
tar -cJf module-debug-bundle.tar.xz debug-directory/
```

Remove secrets before sharing.

### 38. Summary

LSS troubleshooting depends on separating several forms of evidence:

```text
compile log
→ what happened during build

install manifest
→ what LSS finally owns

MD5 log
→ installed checksums

packages
→ current module and policy state

depends
→ current relationship state

activity
→ historical transitions

cache
→ reusable installation artifact
```

The central rule is:

```text
preserve first
→ identify the exact layer
→ diagnose from the correct evidence
→ change one thing at a time

---

## 7. Caches, Archives, and Recovery

### 1. Overview

LSS can preserve reusable installation artifacts after a successful build.

These artifacts are stored under:

```text
/var/cache/lunar
```

They are distinct from:

```text
source archives
build directories
install manifests
compile logs
MD5 logs
package-state records
dependency-state records
```

Understanding these distinctions is essential for recovery, reproducibility, and troubleshooting.

### 2. Main artifact types

A normal module lifecycle may produce or use several artifact classes.

```text
source archive
→ upstream source package

build tree
→ unpacked and compiled working directory

installation cache
→ reusable installed payload

install manifest
→ final paths attributed to the module

compile log
→ build-time output

MD5 log
→ installed checksums

package record
→ current module state

dependency record
→ current relationship state
```

These artifacts may refer to the same module and version, but they serve different purposes.

### 3. Source archives

Source archives contain upstream source code.

They may be downloaded from:

- upstream project sites;
- GitHub or similar hosting;
- Lunar mirrors;
- configured local sources.

Example:

```text
xxHash-0.8.3.tar.gz
```

A source archive is not an installed package cache.

It still requires:

```text
unpack
→ configure
→ compile
→ install
```

unless another reusable artifact is available.

### 4. Build directories

The build directory contains the unpacked and transformed source tree.

It may include:

- generated files;
- object files;
- temporary binaries;
- patched source;
- configure results;
- build-system state.

It is working material, not authoritative installed state.

A build tree may contain files that never reach the system.

It may also lack files generated later during installation.

### 5. Installation caches

When archiving is enabled, LSS may create a cache such as:

```text
/var/cache/lunar/module-version-triplet.tar.xz
```

Observed example:

```text
/var/cache/lunar/xxhash-0.8.3-x86_64-pc-linux-gnu.tar.xz
```

The cache is intended to support installation or resurrection without repeating the full compile path.

### 6. `ARCHIVE`

The global setting:

```text
ARCHIVE=on
```

enables creation of installation archives after successful installation.

Inspect:

```bash
grep -n 'ARCHIVE' /etc/lunar/config
```

Possible value:

```text
ARCHIVE=${ARCHIVE:-on}
```

The effective value may also be influenced by environment or local configuration.

### 7. Inspecting caches

List all caches:

```bash
ls -lh /var/cache/lunar
```

Find one module:

```bash
ls -lh /var/cache/lunar/module-*
```

Example:

```bash
ls -lh /var/cache/lunar/xxhash-*
```

Count matching archives:

```bash
find /var/cache/lunar   -maxdepth 1   -type f   -name 'xxhash-*'   -print
```

### 8. Cache naming

A cache name commonly encodes:

```text
module
version
target triplet
compression
```

Example:

```text
xxhash-0.8.3-x86_64-pc-linux-gnu.tar.xz
```

This helps distinguish:

- version;
- target architecture;
- target system tuple;
- compression format.

It may not encode every meaningful build choice.

### 9. Cache identity is incomplete

Two builds can share:

```text
same module
same version
same target triplet
```

but differ in:

- optional dependencies;
- compiler flags;
- provider choice;
- configure options;
- toolchain version;
- patches;
- local environment.

Therefore:

```text
matching cache filename
≠ guaranteed semantic equivalence
```

A cache may be technically loadable but operationally wrong for the current intended configuration.

### 10. Cache resurrection

Source analysis shows that `lin_module` may attempt cache resurrection before the normal compile path.

Conceptually:

```text
module requested
→ check reusable cache
→ restore if acceptable
→ otherwise continue with source build
```

This can reduce build time significantly.

It can also complicate troubleshooting if the administrator expects a fresh compile.

### 11. Detecting cache use

Possible clues:

- operation completes unusually quickly;
- no normal compiler output appears;
- console mentions recovery or resurrection;
- package state changes without a new compile log;
- cache timestamp precedes installation time;
- installed payload matches an older configuration.

Preserve console output:

```bash
lin module 2>&1 | tee install-console.log
```

Inspect activity:

```bash
grep 'module' /var/log/lunar/activity
```

Inspect compile logs:

```bash
ls -l /var/log/lunar/compile/module-*
```

### 12. Cache versus install manifest

The cache contains reusable installation content.

The install manifest records final ownership.

The relationship is:

```text
cache
→ material that can be restored

manifest
→ paths LSS attributes after installation
```

A cache without a valid manifest is not sufficient for correct removal.

A manifest without a cache is still sufficient for ownership and removal, but not for fast restoration.

### 13. Cache versus package state

A cache can exist even when the module is not currently installed.

A module can be installed even when no cache exists.

Therefore:

```text
cache present
≠ installed

installed
≠ cache present
```

Check current installation through:

```bash
grep '^module:' /var/state/lunar/packages
```

Check cache separately:

```bash
ls -l /var/cache/lunar/module-*
```

### 14. Cache versus compile log

A cache may have been created from an earlier build.

The current installation may have been restored from it.

The compile log may therefore describe:

- the original build;
- a previous installation;
- no current compile at all.

Do not assume the compile-log timestamp always matches the latest installation transaction.

Compare timestamps and activity history.

### 15. Rebuild when a fresh compile is required

A fresh compile is preferable when:

- compiler flags changed;
- toolchain changed;
- optional dependencies changed;
- Moonbase recipe changed;
- patches changed;
- provider changed;
- reproducibility is being tested;
- old cache provenance is uncertain.

In such cases, do not rely blindly on cache restoration.

Preserve or move the existing cache first.

Example:

```bash
mkdir -p /root/lunar-cache-saved

mv /var/cache/lunar/module-version-*   /root/lunar-cache-saved/
```

Then rebuild.

Use a disposable environment for controlled experiments.

### 16. Preserving caches

Copy a module cache:

```bash
mkdir -p /root/lunar-cache-backup

cp -a /var/cache/lunar/module-version-*   /root/lunar-cache-backup/
```

Record checksum:

```bash
sha256sum /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/SHA256SUMS
```

Record metadata:

```bash
stat /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/stat.txt
```

This helps distinguish cache corruption from build problems.

### 17. Inspecting cache contents

For an XZ-compressed tar archive:

```bash
tar -tJf /var/cache/lunar/module-version-triplet.tar.xz
```

For gzip:

```bash
tar -tzf archive.tar.gz
```

For bzip2:

```bash
tar -tjf archive.tar.bz2
```

List before extracting.

Do not extract directly into `/`.

Use a temporary directory:

```bash
mkdir -p /tmp/lunar-cache-inspect

tar -xJf archive.tar.xz   -C /tmp/lunar-cache-inspect
```

### 18. Comparing cache and manifest

Create sorted lists:

```bash
tar -tJf archive.tar.xz |
  sed 's#^\./##' |
  sort > cache.paths

sed 's#^/##' install-manifest |
  sort > manifest.paths
```

Compare:

```bash
diff -u manifest.paths cache.paths
```

Interpret carefully.

The archive may encode paths differently or omit metadata files created outside the cached payload.

### 19. Cache integrity

Verify the archive structure:

```bash
xz -t archive.tar.xz
```

Then verify tar readability:

```bash
tar -tJf archive.tar.xz >/dev/null
```

Checksum:

```bash
sha256sum archive.tar.xz
```

A readable archive can still be semantically stale.

Integrity and suitability are different questions.

### 20. Cache recovery risks

Potential risks:

- wrong optional features;
- stale configuration;
- old toolchain output;
- outdated module recipe;
- incompatible ABI;
- missing post-install side effects;
- old service integration;
- mismatched ownership metadata.

A cache can restore files correctly while still producing the wrong operational system.

### 21. Recovery after accidental removal

Possible recovery sources:

```text
installation cache
source archive
Moonbase module
saved manifest
saved MD5 log
filesystem backup
container snapshot
system backup
```

Preferred recovery order depends on confidence.

If a trusted cache exists:

```text
restore through LSS
→ regenerate state
→ verify ownership
→ verify runtime
```

Avoid manually unpacking a cache into `/` unless performing expert recovery.

Manual extraction may bypass:

- package-state registration;
- dependency-state updates;
- manifest generation;
- configuration policy;
- post-install actions.

### 22. Recovery after failed rebuild

A failed rebuild may leave:

- old package removed;
- partial new payload;
- incomplete manifest;
- stale package record;
- missing logs;
- valid old cache.

Recovery sequence:

```text
preserve current evidence
→ inspect package state
→ inspect manifest
→ inspect cache
→ determine whether old payload remains
→ restore through LSS when possible
→ verify state
```

Do not immediately delete temporary files.

They may explain the failure.

### 23. Recovery after missing manifest

If the package record exists but the install manifest is missing:

```text
installed state
→ present

ownership evidence
→ missing
```

This is dangerous because normal removal cannot reliably know what belongs to the module.

Possible recovery:

1. preserve package and dependency state;
2. inspect caches;
3. rebuild or reinstall the same module configuration;
4. regenerate manifest;
5. compare resulting files;
6. only then consider removal.

Do not invent a manifest manually unless no better recovery path exists.

### 24. Recovery after stale package record

If the package record exists but payload is missing:

```text
packages database
→ says installed

filesystem
→ disagrees
```

Possible causes:

- manual deletion;
- interrupted operation;
- filesystem damage;
- restored state database without payload.

Preferred recovery:

```text
reinstall or rebuild module
→ regenerate payload and logs
→ verify package state
```

Manual database deletion hides the inconsistency without repairing the system.

### 25. Recovery after stale dependency state

If dependency records reference missing modules:

1. preserve `/var/state/lunar/depends`;
2. inspect affected module declarations;
3. inspect package records;
4. rebuild or reconfigure dependent modules;
5. verify new dependency state;
6. remove obsolete records only through supported LSS operations.

The goal is to restore alignment, not merely silence an error.

### 26. Recovery from a container snapshot

For experiments, a container snapshot or backup is often the safest recovery mechanism.

Recommended pattern:

```text
backup container
→ perform experiment
→ preserve evidence
→ restore if necessary
```

This protects the real system while still allowing full lifecycle testing.

### 27. Recovery from system backup

For critical systems, preserve:

```text
/etc
/var/state/lunar
/var/log/lunar
/var/cache/lunar
Moonbase
local configuration
important user data
```

A state-only backup is insufficient if installed payload is lost.

A filesystem-only backup is insufficient if LSS state is lost.

Recovery requires both:

```text
files
+
state
```

### 28. `REAP`

The global setting:

```text
REAP=on
```

controls cleanup behavior.

Inspect:

```bash
grep -n 'REAP' /etc/lunar/config
```

Cleanup may remove temporary build material after success.

This reduces disk use but also removes evidence useful for deep debugging.

For a difficult build problem, consider preserving the build tree before cleanup.

### 29. Preserving a build tree

When a build fails, identify its temporary source/build directory.

Copy it before another run or cleanup:

```bash
cp -a /path/to/build-tree   /root/lss-debug/module-build-tree
```

Also preserve:

```text
configure logs
generated Makefiles
object files
patched sources
command output
```

This can reveal differences not visible in the compressed compile log.

### 30. Archive cleanup

Caches accumulate over time.

Review:

```bash
du -sh /var/cache/lunar
find /var/cache/lunar -maxdepth 1 -type f | wc -l
```

List by size:

```bash
du -h /var/cache/lunar/* | sort -h
```

List by age:

```bash
find /var/cache/lunar   -maxdepth 1   -type f   -printf '%TY-%Tm-%Td %TH:%TM %p\n' |
  sort
```

Do not delete caches solely because they are old.

They may be the fastest recovery path for a difficult package.

### 31. Safe cache cleanup

A careful process:

```text
identify obsolete versions
→ verify current installed version
→ preserve critical caches
→ remove only known-unneeded archives
→ keep recovery path for core modules
```

For one module:

```bash
MODULE=name
CURRENT=$(lvu installed "$MODULE")

ls -l /var/cache/lunar/${MODULE}-*
```

Retain the current trusted cache when possible.

### 32. Core-module caches

Caches for critical modules deserve special value.

Examples:

```text
glibc
gcc
bash
coreutils
lunar
openssl
kernel-related modules
```

A trusted cache can reduce recovery time if a source rebuild becomes impossible.

Preserve checksums and provenance.

### 33. Cache provenance

A useful cache record should include:

```text
module
version
creation date
Moonbase revision
toolchain versions
optional dependency choices
compiler flags
target triplet
checksum
```

The archive filename alone is not enough.

Create sidecar metadata:

```bash
cat > archive.metadata <<EOF
module=...
version=...
moonbase_revision=...
cc=...
cxx=...
cflags=...
created=...
EOF
```

### 34. Reproducible cache sets

For a known system baseline, preserve:

```text
selected caches
packages database
depends database
Moonbase revision
Lunar configuration
checksums
```

This creates a stronger recovery set than arbitrary cache accumulation.

It can support:

- container recreation;
- system restoration;
- regression comparison;
- offline recovery.

### 35. Offline recovery

A useful offline recovery set may contain:

```text
source archives
installation caches
Moonbase
LSS tools
state backups
checksums
documentation
```

Verify the set before it is needed.

A backup that has never been tested is only a hypothesis.

### 36. Recovery validation

After restoration:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
cat /var/log/lunar/install/module-version
```

Verify payload:

```bash
command -v program
ldd /usr/bin/program
```

Verify configuration:

```bash
md5sum /etc/module.conf
```

Verify runtime behavior:

```text
start program
run feature test
check service
check plugin discovery
```

Recovery is complete only when state and runtime agree.

### 37. Cache troubleshooting checklist

If cache restoration gives an unexpected result:

```text
Was the cache actually used?
Does the version match?
Does the target triplet match?
Were options different?
Did Moonbase change?
Did the toolchain change?
Were post-install actions run?
Does the manifest match?
Does runtime behavior match intent?
```

### 38. Common mistakes

### Mistake 1: treating a cache as an installed package record

It is only reusable payload.

### Mistake 2: trusting filename identity

It may omit important build choices.

### Mistake 3: manually unpacking into `/`

This bypasses LSS state management.

### Mistake 4: deleting all old caches

This removes valuable recovery paths.

### Mistake 5: keeping caches without provenance

Their suitability becomes difficult to assess.

### Mistake 6: repairing only state or only files

Recovery requires alignment between both.

### Mistake 7: assuming archive integrity means configuration correctness

A valid archive can still be stale.

### 39. Safe recovery model

```text
preserve evidence
→ identify the damaged layer
→ select trusted recovery artifact
→ restore through LSS
→ regenerate state
→ verify ownership
→ verify runtime
```

### 40. Summary

LSS caches and archives support fast restoration, but they are not substitutes for state.

The full recovery model is:

```text
source
→ rebuild capability

cache
→ reusable installed payload

manifest
→ ownership

MD5 log
→ installed checksums

packages
→ current module state

depends
→ relationship state

backup
→ broader restoration
```

The central rule is:

```text
treat cache as evidence and reusable material
→ not as complete system truth

---

## 8. Updating the System Safely

### 1. Overview

Updating a source-based system is not a single file replacement.

An update may affect:

- module definitions;
- dependency choices;
- toolchain behavior;
- ABI compatibility;
- installed manifests;
- caches;
- local configuration;
- service integration;
- future rebuilds.

A safe update process is therefore:

```text
inspect
→ plan
→ preserve state
→ update in dependency order
→ verify
→ expand gradually
```

### 2. Types of update

Common update types include:

```text
single module version update
single module rebuild
dependency update
provider replacement
toolchain update
library ABI transition
Moonbase update
large system rebuild
kernel or boot component update
LSS self-update
```

Each has a different risk profile.

### 3. Small versus systemic updates

A small leaf utility update usually has a limited blast radius.

A systemic update may affect many modules.

High-impact examples:

```text
glibc
gcc
binutils
llvm
openssl
zlib
python
perl
pkgconf
lunar
kernel
```

Treat toolchain and core-library transitions as separate engineering operations.

### 4. Before updating Moonbase

Record the current state:

```bash
cp -a /var/state/lunar/packages   /root/packages.before-moonbase

cp -a /var/state/lunar/depends   /root/depends.before-moonbase
```

Record the active Moonbase revision if it is managed by Git:

```bash
git -C /var/lib/lunar/moonbase rev-parse HEAD   > /root/moonbase-revision.before
```

Preserve local modifications:

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

Do not update over unreviewed local changes.

### 5. Inspecting available updates

Compare installed and Moonbase versions.

For one module:

```bash
printf 'installed: %s
' "$(lvu installed module)"
printf 'moonbase:  %s
' "$(lvu version module)"
```

For several modules:

```bash
for module in module1 module2 module3; do
  printf '%-24s installed=%-16s moonbase=%s
'     "$module"     "$(lvu installed "$module" 2>/dev/null)"     "$(lvu version "$module" 2>/dev/null)"
done
```

A version difference indicates a candidate update, not automatic safety.

### 6. Read the module change

Before updating, inspect:

- `DETAILS`;
- `BUILD`;
- `DEPENDS`;
- `CONFIGURE`;
- `OPTIONS`;
- lifecycle hooks;
- patches.

Compare old and new Moonbase revisions when possible:

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   section/module
```

Look for:

```text
version change
source URL change
checksum change
new dependency
removed dependency
new build option
path layout change
service change
remove hook
toolchain requirement
```

### 7. Preserve the current module state

For a module:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-update/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

For critical modules, preserve the cache and relevant configuration too.

### 8. Dependency order

Updates should respect dependency direction.

Conceptually:

```text
dependency
→ update first

dependent
→ rebuild or update afterward
```

A library update may require rebuilding consumers even when their version does not change.

Use:

```bash
lvu depends library
lvu leert library
```

to estimate reverse impact.

### 9. Topological ordering

LSS uses dependency graph ordering for multi-module operations.

The general principle is:

```text
build prerequisites first
→ then modules that consume them
```

Avoid manual arbitrary order when several linked modules are involved.

### 10. Single-module update

A controlled single-module update:

```bash
lin -c module 2>&1 |
  tee /root/lss-update/module/rebuild.log
```

Afterward:

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
```

Compare manifests and dependency state.

Test the actual program or library.

### 11. Updating a shared library

A shared library update may preserve the same SONAME or change it.

Check before:

```bash
readelf -d /usr/lib/libexample.so | grep SONAME
```

Check consumers:

```bash
lvu depends library
lvu leert library
```

After update:

```bash
ldconfig
ldd /usr/bin/consumer | grep 'not found'
```

A successful library build does not prove all consumers remain functional.

### 12. ABI transitions

An ABI transition may require rebuilding reverse dependents.

Warning signs:

```text
SONAME changed
major version changed
symbols removed
compiler runtime changed
C++ standard library changed
language runtime changed
```

Safe sequence:

```text
update library
→ verify installed library
→ rebuild direct consumers
→ test
→ rebuild indirect consumers as needed
```

### 13. Toolchain updates

Toolchain transitions deserve special handling.

Components may include:

```text
binutils
gcc
glibc
llvm
clang
lld
compiler-rt
libc++
make
pkgconf
```

A toolchain update can affect every later build.

Record before:

```bash
gcc --version
clang --version
ld --version
make --version
```

Record relevant variables:

```bash
env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS)='
```

### 14. Validate toolchain on a small module

Before rebuilding core components, test a small known module.

Good validation characteristics:

```text
small
quick to build
simple C or C++
clear manifest
easy runtime test
```

Verify:

- compilation;
- linking;
- installwatch;
- manifest;
- runtime.

Only then expand.

### 15. Compiler-family consistency

Do not mix compiler-family assumptions carelessly.

Example problem:

```text
BUILD expects Clang
→ environment uses GCC
→ Clang-specific flags fail
```

or the reverse.

For LLVM-family modules, explicit selection may be required:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

The correct choice belongs in module or toolchain policy, not ad hoc repeated fixes.

### 16. Rebuilding reverse dependents

After a library or toolchain change, identify consumers:

```bash
lvu depends target
lvu leert target
```

Start with direct consumers.

Rebuild in waves:

```text
wave 1
→ direct dependents

wave 2
→ higher-level consumers

wave 3
→ applications and optional integrations
```

Verify after each wave.

### 17. Optional dependency changes during update

A Moonbase update may introduce new optional dependencies.

Do not automatically enable all of them.

Review:

```text
what feature does it add?
is it needed?
does it enlarge the dependency graph?
does it introduce a service or provider?
does it change attack surface?
```

Preserve administrator intent.

### 18. Required dependency changes

A new required dependency changes the minimum module form.

Before accepting:

1. inspect the dependency;
2. inspect its own dependencies;
3. inspect conflicts;
4. estimate build and runtime impact;
5. record the decision.

A required dependency is part of the updated module contract.

### 19. Configuration drift during updates

A newer module recipe may interpret old choices differently.

Possible drift:

```text
old option removed
new default introduced
provider renamed
optional dependency becomes required
configure auto-detection changes
```

After update, compare:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
diff old-install-log new-install-log
```

Test intended features.

### 20. Preserve `/etc`

Before updating a service or daemon:

```bash
cp -a /etc/module.conf   /root/module.conf.before-update
```

Afterward:

```bash
diff -u   /root/module.conf.before-update   /etc/module.conf
```

Also inspect MD5 logs.

A rebuild may preserve, replace, merge, or leave configuration unchanged depending on module and upstream behavior.

### 21. Services

After updating a service:

```text
verify binary
→ verify configuration
→ restart or reload
→ inspect logs
→ test client behavior
```

Check service-manager state using the system's actual init framework.

Installation success is not service validation.

### 22. Kernel and boot updates

Kernel, initramfs, bootloader, and firmware updates are high risk.

Before:

- ensure a known-good kernel remains available;
- preserve boot configuration;
- preserve initramfs;
- verify filesystem space;
- verify recovery media;
- avoid deleting the previous working entry immediately.

After:

- verify generated files;
- verify bootloader entries;
- reboot only after inspection;
- retain fallback.

### 23. LSS self-update

Updating `lunar` or core LSS components can affect all future operations.

Before:

```bash
cp -a /var/lib/lunar/functions   /root/lunar-functions.before

cp -a /sbin/lin /sbin/lrm /bin/lvu   /root/lunar-programs.before/

cp -a /etc/lunar   /root/lunar-config.before
```

Use a container or recovery environment when testing major LSS changes.

### 24. Multi-module updates

For a planned set:

```text
group by dependency layer
→ update low-level components first
→ rebuild consumers
→ test after each layer
```

Do not combine unrelated high-risk transitions into one run.

Bad pattern:

```text
toolchain update
+ libc update
+ init update
+ package manager update
```

all at once.

This makes failure attribution difficult.

### 25. Update waves

A practical update strategy:

### Wave 1: low-risk leaves

Small utilities and isolated modules.

### Wave 2: common libraries

Shared libraries with manageable reverse dependency sets.

### Wave 3: applications

User-facing programs.

### Wave 4: services

Daemons and network-facing components.

### Wave 5: core and toolchain

Only after earlier validation.

### 26. Checkpoints between waves

After each wave, record:

```bash
cp -a /var/state/lunar/packages   packages.wave-N

cp -a /var/state/lunar/depends   depends.wave-N
```

Run basic tests.

Do not continue if state is inconsistent.

### 27. Disk space

Source builds require space for:

```text
source archive
unpacked tree
object files
installed payload
logs
cache archive
old version during transition
```

Check:

```bash
df -h
du -sh /var/cache/lunar
du -sh /var/log/lunar
```

A full filesystem during installation can leave partial state.

### 28. Source availability

Before a critical update, verify source access.

Potential problems:

```text
upstream URL removed
tag renamed
mirror unavailable
checksum changed
network failure
```

Preserve source archives for core modules.

### 29. Cache use during updates

A cache may accelerate recovery.

But after configuration, Moonbase, or toolchain changes, it may restore an old result.

For a controlled fresh update:

```text
preserve old cache
→ prevent accidental reuse
→ rebuild
→ create new cache
→ compare
```

### 30. Failed update response

If an update fails:

```text
stop expansion
→ preserve evidence
→ identify failed phase
→ inspect current package state
→ determine whether old payload remains
→ restore from trusted cache or rebuild
```

Do not immediately update other modules.

### 31. Partial installation detection

Check:

```bash
grep '^module:' /var/state/lunar/packages
ls -l /var/log/lunar/install/module-*
ls -l /var/log/lunar/md5sum/module-*
command -v program
```

Possible states:

```text
old record + old payload
new record + new payload
no record + partial payload
record present + missing manifest
```

Each requires a different recovery action.

### 32. Rollback

A rollback may use:

- trusted old cache;
- old source and Moonbase recipe;
- filesystem snapshot;
- container backup;
- system backup;
- previous kernel or boot entry.

Rollback should restore:

```text
payload
ownership manifest
MD5 log
package state
dependency state
configuration
```

Restoring only the binary is incomplete.

### 33. Verifying after update

### State

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
```

### Ownership

```bash
cat /var/log/lunar/install/module-version
```

### Dependencies

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

### Linking

```bash
ldd /usr/bin/program | grep 'not found'
```

### Configuration

```bash
diff -u config.before /etc/module.conf
```

### Runtime

Run an actual feature or service test.

### 34. System-wide consistency checks

After a larger update:

```bash
find /var/log/lunar/install   -maxdepth 1   -type f   | sort
```

Look for obvious missing state.

Check broken links:

```bash
find /usr -xtype l -print 2>/dev/null
```

Check missing shared libraries in important binaries.

Avoid indiscriminate full-filesystem scans during normal operation unless needed.

### 35. Activity review

Review recent transitions:

```bash
tail -200 /var/log/lunar/activity
```

Search failures:

```bash
grep -i 'failed' /var/log/lunar/activity | tail -50
```

Compare with console and compile logs.

### 36. Update documentation

For significant updates, record:

```text
date
modules
old versions
new versions
Moonbase revision
dependency changes
configuration changes
toolchain
tests performed
problems
recovery actions
```

This becomes valuable evidence for future maintenance.

### 37. Safe update checklist

Before:

```text
review Moonbase change
inspect dependencies
save state and logs
verify disk space
verify source availability
preserve configuration
prepare rollback
```

During:

```text
update in dependency order
change one major layer at a time
preserve console output
stop on unexplained failure
```

After:

```text
verify packages and depends
compare manifests
check linking
check configuration
test runtime
create checkpoint
```

### 38. Common mistakes

### Mistake 1: updating everything at once

Failure attribution becomes difficult.

### Mistake 2: ignoring reverse dependents

Library updates can break consumers.

### Mistake 3: trusting version numbers alone

Configuration and ABI may have changed.

### Mistake 4: deleting the old cache immediately

It may be the best rollback path.

### Mistake 5: rebuilding core modules before validating the toolchain

Test a small module first.

### Mistake 6: assuming build success means system success

Runtime and service verification remain necessary.

### Mistake 7: updating over local Moonbase changes

Preserve and review them first.

### 39. Recommended update model

```text
observe current system
→ preserve state
→ review module changes
→ update dependencies first
→ rebuild consumers
→ verify each wave
→ preserve rollback until confidence is high
```

### 40. Summary

Safe system updating in LSS is controlled graph evolution.

The administrator is not only replacing versions.

They are managing:

```text
source
dependencies
configuration
ABI
ownership
persistent state
runtime behavior
recovery paths
```

The central rule is:

```text
small validated steps
→ explicit checkpoints
→ gradual expansion

---

## 9. Working with Moonbase

### 1. Overview

Moonbase is the active collection of module definitions used by LSS.

It tells LSS:

- which modules exist;
- where their sources are;
- which versions are available;
- how software is built;
- which dependencies are required or optional;
- which hooks and policies apply.

Moonbase is not the installed system.

It is the source of module intent.

The installed system is represented separately through:

```text
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/md5sum
```

### 2. Active Moonbase location

A common active path is:

```text
/var/lib/lunar/moonbase
```

Inspect:

```bash
ls -la /var/lib/lunar/moonbase
```

If managed by Git:

```bash
git -C /var/lib/lunar/moonbase status
git -C /var/lib/lunar/moonbase rev-parse HEAD
```

The active Moonbase used by the current system is more relevant than an unrelated repository clone.

### 3. Module sections

Modules are organized into sections.

Examples:

```text
core/libs
core/filesys
core/utils
```

A module name remains the normal operational identifier:

```bash
lin foremost
lrm foremost
lvu where foremost
```

The section is used to locate its definition.

### 4. Locating a module

Use:

```bash
lvu where module
```

Example:

```bash
lvu where xxhash
```

Possible output:

```text
core/libs
```

Construct the full path:

```bash
MODULE=xxhash
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s\n' "$MODULE_DIR"
```

### 5. Listing sections and modules

To inspect a section:

```bash
lvu section section-name
```

Use `lvu where` for a module name.

Do not confuse:

```bash
lvu section xxhash
```

with:

```bash
lvu where xxhash
```

The first expects a section.

The second locates a module.

### 6. Moonbase as executable policy

A Moonbase module is not static metadata.

Its files contain executable shell logic.

Examples:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

Therefore Moonbase acts as:

```text
software catalog
+
build policy
+
dependency policy
+
lifecycle policy
```

### 7. Active Moonbase versus upstream repository

The active Moonbase may differ from upstream.

Possible reasons:

- local modifications;
- uncommitted patches;
- delayed update;
- pinned revision;
- private modules;
- experimental changes;
- branch differences.

Always inspect the active copy before diagnosing current behavior.

Upstream is useful for comparison, not automatic proof of what the local system executed.

### 8. Recording Moonbase revision

Before a significant update:

```bash
git -C /var/lib/lunar/moonbase rev-parse HEAD   > /root/moonbase-revision.before
```

Record branch:

```bash
git -C /var/lib/lunar/moonbase branch --show-current   > /root/moonbase-branch.before
```

Record status:

```bash
git -C /var/lib/lunar/moonbase status --short   > /root/moonbase-status.before
```

This makes later build results traceable.

### 9. Detecting local changes

Use:

```bash
git -C /var/lib/lunar/moonbase status --short
```

Inspect diff:

```bash
git -C /var/lib/lunar/moonbase diff
```

Save patch:

```bash
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

Do not update Moonbase until local changes are understood and preserved.

### 10. Why local changes matter

A one-line change can alter:

- compiler selection;
- dependency graph;
- source URL;
- checksum;
- installed paths;
- service behavior;
- remove hooks;
- configuration defaults.

A build may therefore differ even when module name and version are unchanged.

### 11. Comparing a module across revisions

For one module:

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   core/libs/xxhash
```

Inspect history:

```bash
git -C /var/lib/lunar/moonbase log --   core/libs/xxhash
```

Show a previous version:

```bash
git -C /var/lib/lunar/moonbase show   REVISION:core/libs/xxhash/BUILD
```

This helps identify packaging changes separately from upstream changes.

### 12. Updating Moonbase safely

A safe process:

```text
record revision
→ preserve local changes
→ update Moonbase
→ inspect changed modules
→ review dependency changes
→ update selected modules gradually
```

Do not treat Moonbase update and full-system rebuild as the same operation.

Updating definitions does not automatically rebuild installed software.

### 13. Reviewing changed module definitions

After update:

```bash
git -C /var/lib/lunar/moonbase diff   REVISION_BEFORE..HEAD
```

List changed files:

```bash
git -C /var/lib/lunar/moonbase diff   --name-only REVISION_BEFORE..HEAD
```

Focus on:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
hooks
patches
```

### 14. Module version versus packaging revision

A module can change without changing upstream version.

Examples:

- patch added;
- dependency corrected;
- compiler flag changed;
- install path fixed;
- checksum method changed;
- hook adjusted.

Therefore:

```text
same VERSION
≠ identical module behavior
```

Moonbase revision is part of reproducibility.

### 15. Local modules

A local module may be used for:

- private software;
- experimental updates;
- hardware-specific tools;
- unreleased patches;
- local services;
- testing new packaging.

Keep local modules clearly separated from upstream-managed content when possible.

This reduces accidental overwrite and makes ownership clear.

### 16. Local section strategy

A practical approach is a dedicated local section.

Conceptually:

```text
local/tools
local/services
local/libs
```

Benefits:

- easier review;
- easier backup;
- reduced merge conflict;
- clearer provenance;
- safer upstream updates.

Follow the actual Moonbase and LSS conventions in use on the system.

### 17. Testing a local module

Use a disposable environment.

Workflow:

```text
create module
→ inspect shell logic
→ build in container
→ inspect console output
→ inspect manifest
→ inspect packages and depends
→ test runtime
→ remove
→ verify cleanup
```

Do not begin with a critical production host.

### 18. Source verification

`DETAILS` may define:

- source archive;
- source URL;
- verification data;
- upstream site.

Inspect before trusting a new or modified module.

Check that:

```text
source URL matches intended upstream
version matches archive
verification data matches source
description matches software
```

A source URL change deserves review.

### 19. Patches

Patches are part of Moonbase behavior.

Inspect:

```bash
find "$MODULE_DIR" -type f   \( -name '*.patch' -o -name '*.diff' \)   -print
```

Search usage:

```bash
grep -R -n -E 'patch|sedit' "$MODULE_DIR"
```

A patch file present but unused does not affect the build.

A patch applied conditionally may affect only some configurations.

### 20. Dependency review after Moonbase update

Compare `DEPENDS` changes.

Potential impacts:

```text
new required dependency
new optional feature
removed dependency
provider replacement
default on/off change
```

Inspect current stored state:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
```

The declaration may have changed while the installed state still reflects the old configuration.

### 21. Configuration review after update

A changed `CONFIGURE` or `OPTIONS` file may:

- add a new question;
- remove an old option;
- change defaults;
- rename a feature;
- alter provider selection.

Before rebuild, determine whether old stored choices still map correctly.

### 22. Build review after update

A changed `BUILD` may alter:

- toolchain;
- configure flags;
- install prefix;
- static/shared library policy;
- documentation;
- cleanup;
- generated files.

Compare old and new `BUILD` directly.

### 23. Hook review after update

New or changed hooks deserve special attention.

Examples:

```text
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

A new hook can introduce side effects beyond ordinary ownership handling.

### 24. Moonbase and installed state can diverge

Common divergence:

```text
Moonbase version newer than installed
installed module built from old recipe
local Moonbase modified after installation
dependency declaration changed
current manifest reflects previous recipe
```

This is normal until the module is rebuilt or updated.

Do not expect the installed system to change immediately when Moonbase changes.

### 25. Inspecting installed versus current module definition

Record:

```bash
MODULE=name
VERSION_INSTALLED=$(lvu installed "$MODULE")
VERSION_MOONBASE=$(lvu version "$MODULE")
SECTION=$(lvu where "$MODULE")
```

Show:

```bash
printf 'installed=%s\n' "$VERSION_INSTALLED"
printf 'moonbase=%s\n' "$VERSION_MOONBASE"
printf 'section=%s\n' "$SECTION"
```

Then inspect history to determine whether the recipe changed.

### 26. Pinning Moonbase revision

For a stable environment, pinning a known revision may be useful.

Benefits:

- reproducible module definitions;
- controlled updates;
- easier rollback;
- clearer evidence.

Costs:

- delayed fixes;
- delayed security updates;
- manual update planning.

A pinned revision must be documented.

### 27. Branches

Different branches may represent:

- stable;
- testing;
- development;
- local experimentation.

Check:

```bash
git -C /var/lib/lunar/moonbase branch --show-current
```

Do not switch branches on a production system without reviewing differences.

### 28. Updating with local modifications

Safe options:

```text
commit local changes
stash local changes
export patch
use dedicated local branch
move private modules to local section
```

Avoid:

```text
pull and hope
```

A merge conflict inside `BUILD` or `DEPENDS` can produce subtle breakage.

### 29. Moonbase backup

Backup:

```bash
tar -cJf /root/moonbase-backup.tar.xz   /var/lib/lunar/moonbase
```

Or preserve Git metadata and working tree through filesystem backup.

For a compact reproducibility record, save:

```text
revision
branch
status
local diff
selected module directories
```

### 30. Searching Moonbase

Search module names:

```bash
find /var/lib/lunar/moonbase   -mindepth 2   -maxdepth 3   -type d   -name 'pattern'
```

Search content:

```bash
grep -R -n -E 'prepare_install|optional_depends|CC='   /var/lib/lunar/moonbase
```

Use narrow searches to avoid excessive noise.

### 31. Finding modules using a dependency

Search persistent state:

```bash
grep -E '^[^:]+:dependency:'   /var/state/lunar/depends
```

Search declarations:

```bash
grep -R -n -F 'dependency'   /var/lib/lunar/moonbase/*/*/*/DEPENDS
```

These answer different questions:

```text
persistent state
→ currently resolved relationships

Moonbase search
→ declared possibilities
```

### 32. Finding modules with hooks

```bash
find /var/lib/lunar/moonbase   -type f   \( -name PRE_REMOVE -o -name POST_REMOVE      -o -name PRE_BUILD -o -name POST_BUILD      -o -name POST_INSTALL \)   -print
```

Useful for architectural review and risk analysis.

### 33. Finding modules that install plugins

Search:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Review such modules carefully before removal or update.

### 34. Canonical source versus working fork

For project research, distinguish:

```text
canonical upstream repository
working fork
local active Moonbase
```

The canonical repository explains upstream intent.

The working fork explains local development.

The active Moonbase explains current system behavior.

Never collapse these into one source.

### 35. Reproducibility record

For a significant build, preserve:

```bash
OUT=/root/lss-build-record

mkdir -p "$OUT"

git -C /var/lib/lunar/moonbase rev-parse HEAD   > "$OUT/moonbase-revision"

git -C /var/lib/lunar/moonbase status --short   > "$OUT/moonbase-status"

git -C /var/lib/lunar/moonbase diff   > "$OUT/moonbase-diff"

cp -a "$MODULE_DIR" "$OUT/module-definition"
```

Add package, dependency, environment, and toolchain records.

### 36. Common mistakes

### Mistake 1: diagnosing from upstream only

The active local Moonbase may differ.

### Mistake 2: updating over local modifications

This risks lost or merged behavior.

### Mistake 3: assuming Moonbase update changes installed software

A rebuild is still required.

### Mistake 4: comparing only VERSION

Packaging logic can change independently.

### Mistake 5: mixing local modules into upstream sections without documentation

This complicates updates and provenance.

### Mistake 6: treating a fork as canonical

Use the correct source for the question.

### Mistake 7: switching branches without reviewing module differences

This can change the entire build policy.

### 37. Safe Moonbase workflow

```text
inspect active revision
→ preserve local changes
→ update
→ review diffs
→ classify changed modules
→ rebuild selectively
→ verify state and runtime
```

### 38. Summary

Moonbase is the active executable definition of the software universe known to LSS.

It describes:

```text
identity
source
dependencies
configuration
build
installation
removal
integration
```

The central rule is:

```text
Moonbase defines intent
→ installed state records result
→ runtime verifies reality

---

## 10. Advanced Module Inspection and Debugging

### 1. Overview

Most LSS problems can be solved with ordinary inspection:

```text
console output
compile log
install manifest
MD5 log
packages
depends
Moonbase module files
```

Advanced debugging is needed when these views disagree or when the normal evidence is incomplete.

Typical cases include:

- package record exists but files are missing;
- manifest exists but package record is absent;
- dependency state looks stale;
- rebuild changes metadata unexpectedly;
- a file appears during build but not in the final manifest;
- removal preserves or deletes an unexpected path;
- cache behavior hides the actual build path;
- a hook produces side effects outside the manifest.

The purpose of advanced debugging is not to collect more data blindly.

It is to isolate one unclear transition and observe it precisely.

### 2. The advanced debugging model

A useful model is:

```text
declared intent
→ runtime events
→ interpreted ownership
→ persistent state
→ physical filesystem
→ runtime behavior
```

These layers may disagree.

Advanced inspection compares them directly.

### 3. Preserve first

Before changing anything:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-advanced/$MODULE

mkdir -p "$BASE/before" "$BASE/after"
```

Preserve state:

```bash
cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true
```

Preserve logs:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

Preserve module definition:

```bash
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

cp -a "$MODULE_DIR" "$BASE/before/module-definition"
```

### 4. Record the environment

Record the execution environment:

```bash
uname -a > "$BASE/before/uname"

env | sort > "$BASE/before/environment"

{
  command -v gcc && gcc --version
  command -v clang && clang --version
  command -v ld && ld --version
  command -v make && make --version
} > "$BASE/before/toolchain" 2>&1
```

Also record global LSS configuration:

```bash
cp -a /etc/lunar/config   "$BASE/before/lunar-config"
```

Environment drift can explain behavior that module and state files cannot.

### 5. Build a precise question

Good advanced-debugging questions are narrow.

Examples:

```text
Why is this file missing from the final manifest?

Why did installed size change while the manifest stayed identical?

Why did lrm preserve this /etc file?

Why did dependency state remain unchanged after rebuild?

Was this module compiled or restored from cache?

Which exact command removed this path?
```

Avoid broad questions such as:

```text
Why is LSS broken?
```

A precise question determines what evidence to capture.

### 6. Before/after filesystem state

Use the existing install manifest as the scope.

```bash
MANIFEST="$BASE/before/install-log"

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link	%s	%s
' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file	%s
' "$path"
  elif [ -d "$path" ]; then
    printf 'directory	%s
' "$path"
  else
    printf 'missing	%s
' "$path"
  fi
done < "$MANIFEST"   > "$BASE/before/filesystem-state"
```

Repeat after the operation.

Compare:

```bash
diff -u   "$BASE/before/filesystem-state"   "$BASE/after/filesystem-state"
```

This shows physical transitions for known owned paths.

### 7. Before/after persistent state

After the operation:

```bash
cp -a /var/state/lunar/packages   "$BASE/after/packages"

cp -a /var/state/lunar/depends   "$BASE/after/depends"
```

Compare:

```bash
diff -u   "$BASE/before/packages"   "$BASE/after/packages"

diff -u   "$BASE/before/depends"   "$BASE/after/depends"
```

This is often more useful than summarizing records manually.

It reveals exact database mutations.

### 8. Before/after manifests

Preserve the new manifest:

```bash
VERSION_AFTER=$(lvu installed "$MODULE")

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

Compare:

```bash
diff -u   "$BASE/before/install-log"   "$BASE/after/install-log"
```

Possible results:

```text
no difference
→ same final ownership set

new paths
→ new feature or packaging change

removed paths
→ disabled feature or packaging change

reordered paths only
→ generation-order difference
```

Sort only when order itself is not under investigation.

### 9. Raw installwatch evidence

Installwatch records filesystem operations during the installation phase.

The raw stream may include:

- open;
- chmod;
- mkdir;
- link;
- symlink;
- rename;
- unlink.

It is normally temporary and destroyed after the lifecycle completes.

Capturing it requires controlled instrumentation.

### 10. When raw installwatch is useful

Use raw capture when:

```text
a file appears in build output but not in the manifest
a path is created and later removed
ownership filtering is unclear
rename or unlink order matters
a transient file changes build behavior
```

Do not capture raw installwatch for routine administration.

### 11. Safe raw-capture environment

Use:

- disposable container;
- chroot;
- test VM;
- snapshot-capable system;
- verified backup.

Before patching LSS:

```bash
cp -a /var/lib/lunar/functions/main.lunar   /root/main.lunar.before
```

Restore it immediately after the experiment.

### 12. Minimal instrumentation principle

Instrument only the needed boundary.

For example, copy `INSTALLWATCHFILE` immediately before its normal destruction.

Conceptually:

```bash
if [ -f "$INSTALLWATCHFILE" ]; then
  cp "$INSTALLWATCHFILE" /controlled/evidence/path
  temp_destroy "$INSTALLWATCHFILE"
fi
```

Do not alter parsing, lifecycle order, or return values.

The instrumentation should observe, not redesign behavior.

### 13. Raw stream interpretation

A raw sequence such as:

```text
open    /usr/lib/libexample.a
chmod   /usr/lib/libexample.a
unlink  /usr/lib/libexample.a
```

means:

```text
file created or opened during install
→ permissions set
→ file removed later
```

If the path is absent from the final manifest, that is consistent with final-state ownership.

### 14. Raw events are not ownership

Important distinction:

```text
raw installwatch event
→ something happened

final manifest entry
→ LSS claims final ownership
```

A successful event does not guarantee final ownership.

A missing event does not automatically prove no ownership if another path or mechanism was used.

Interpret raw events alongside BUILD and POST_BUILD.

### 15. Investigating installed-size changes

If package size changes unexpectedly:

1. compare old and new manifests;
2. reproduce current size calculation;
3. classify manifest entries;
4. check shell aliases;
5. inspect changed regular files;
6. inspect Lunar logs included in the manifest.

Reproduce:

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    printf '%8s KB  %s
' "$value" "$path"
    SIZE=$((SIZE + value))
  fi
done < install-log

echo "TOTAL=${SIZE}KB"
```

Use `du -k` explicitly.

### 16. Investigating missing paths

If a manifest path is missing:

```bash
grep -Fx '/path' "$BASE/before/install-log"
```

Search current manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Inspect module source:

```bash
grep -R -n -F '/path' "$MODULE_DIR"
```

Check whether a service recreated or removed it.

### 17. Investigating overlapping ownership

If several manifests contain the same path:

```bash
grep -R -F -x '/path'   /var/log/lunar/install
```

Possible explanations:

- provider transition;
- intentional shared path;
- stale manifest;
- historical packaging error;
- replacement module;
- common generated file.

Do not remove or reassign manually before understanding the relationship.

### 18. Investigating `/etc` behavior

For an unexpected preserved or removed configuration:

```bash
md5sum /etc/example.conf
grep '/etc/example.conf' saved-md5sum-log
grep -n 'PRESERVE' /etc/lunar/config
```

Interpret:

```text
checksum same
→ unchanged at removal time

checksum different
→ locally modified

PRESERVE=on
→ modified file should remain
```

Also inspect `PROTECTED` policy and remove hooks.

### 19. Investigating dependency-state drift

Capture raw records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Inspect declaration:

```bash
sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Inspect configuration:

```bash
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"

test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

Possible drift:

```text
stored choice from old module definition
new declaration not yet applied
manual database change
interrupted reconfiguration
provider renamed
optional dependency default changed
```

### 20. Investigating cache use

Preserve console output:

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"
```

Inspect:

```bash
ls -l /var/cache/lunar/${MODULE}-*
ls -l /var/log/lunar/compile/${MODULE}-*
grep "$MODULE" /var/log/lunar/activity
```

A missing fresh compile path may indicate cache resurrection.

Move the cache aside for a controlled fresh build.

### 21. Investigating hooks

Hooks can affect state outside the manifest.

Inspect:

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  echo "=== $hook ==="
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

Search external paths and commands.

A hook may:

- stop services;
- rebuild indexes;
- remove generated data;
- create users or groups;
- write outside normal ownership paths.

### 22. Controlled single-variable experiments

Change only one factor:

```text
compiler
dependency choice
cache availability
Moonbase revision
PRESERVE setting
module patch
```

Bad experiment:

```text
new compiler
+ new Moonbase
+ new options
+ removed cache
```

A result from several simultaneous changes is difficult to interpret.

### 23. Rebuild versus reinstall versus removal

Choose the correct transition.

```text
rebuild
→ inspect upgrade-style replacement

remove
→ inspect ownership cleanup

install after removal
→ inspect fresh installation

cache restoration
→ inspect resurrection path
```

Do not mix transitions in one evidence bundle without clear phase boundaries.

### 24. Evidence naming

Use explicit names:

```text
packages.before
packages.after
depends.before
depends.after
install-log.before
install-log.after
console.log
installwatch.raw
filesystem-state.before
filesystem-state.after
```

Avoid vague names such as:

```text
test1
new
final2
```

Clear names reduce interpretation errors.

### 25. Evidence immutability

Once captured, do not overwrite raw evidence.

Use new files for interpretation:

```text
raw/
derived/
notes/
```

Example:

```text
raw/installwatch.raw
raw/packages.before
derived/installwatch-summary.txt
notes/conclusions.md
```

Raw evidence should remain unchanged.

### 26. Timestamp discipline

Record exact date and time:

```bash
date --iso-8601=seconds   > "$BASE/timestamp"
```

Record timezone if evidence may be compared across systems.

LSS state dates use compact forms such as:

```text
YYYYMMDD
```

Console and filesystem timestamps may use different time formats.

### 27. Checksums for evidence

For a completed evidence directory:

```bash
find "$BASE" -type f -print0 |
  sort -z |
  xargs -0 sha256sum   > "$BASE/SHA256SUMS"
```

Store the checksum file outside or regenerate carefully to avoid self-reference.

Checksums help verify that raw evidence was not altered.

### 28. Exporting from a container

Example:

```bash
podman cp   lunar-dev:/root/lss-evidence/.   ~/LSS-Evidence/
```

Verify:

```bash
find ~/LSS-Evidence -type f | sort
```

Do not destroy the test container before confirming the export.

### 29. Restoring instrumentation

After testing:

```bash
cp -a /root/main.lunar.before   /var/lib/lunar/functions/main.lunar
```

Verify the exact source fragment.

Do not assume restoration succeeded because the copy command returned no error.

### 30. Restoring test modules

After a removal experiment:

```text
remove orphaned test configuration if appropriate
→ reinstall module
→ verify package record
→ verify canonical configuration
→ verify runtime
```

Return the environment to a known clean state.

### 31. Advanced inconsistency matrix

### Package record present, manifest present, payload missing

Likely:

- manual deletion;
- filesystem damage;
- partial replacement.

Preferred action:

```text
preserve
→ rebuild/reinstall
→ verify
```

### Package record present, manifest missing

Likely:

- state/log inconsistency;
- manual log deletion.

Preferred action:

```text
preserve
→ rebuild same configuration
→ regenerate manifest
```

### Package record absent, manifest present

Likely:

- interrupted removal;
- manual database edit;
- stale logs.

Preferred action:

```text
preserve
→ inspect activity
→ determine real payload ownership
```

### Depends record references missing module

Likely:

- stale dependency state;
- interrupted reconfiguration;
- manual removal.

Preferred action:

```text
reconfigure/rebuild dependent module
```

### 32. Physical reality checks

Check binary:

```bash
command -v program
file /usr/bin/program
```

Check linkage:

```bash
ldd /usr/bin/program
```

Check library:

```bash
readelf -d /usr/lib/libexample.so
```

Check service:

```text
service state
process state
listening socket
functional request
```

Persistent state is not a substitute for runtime validation.

### 33. Advanced troubleshooting report

A strong report contains:

```text
problem statement
expected behavior
actual behavior
environment
module version
Moonbase revision
configuration
dependency state
package state
console output
compile log
manifest
MD5 log
raw evidence when needed
before/after diffs
conclusion
remaining uncertainty
```

This makes review efficient.

### 34. Avoiding false conclusions

Common false conclusions:

```text
file absent from manifest
→ file was never created

no depends record
→ no operational dependency

same version
→ same build

successful lin
→ runtime feature works

preserved config
→ package still owns it

cache archive valid
→ cache matches intended configuration
```

Each requires a separate verification step.

### 35. When not to instrument

Do not instrument when:

- normal logs already answer the question;
- the system is production-critical;
- no rollback exists;
- the experiment has no precise hypothesis;
- the code path is poorly understood;
- evidence cannot be exported safely.

Advanced instrumentation should reduce uncertainty, not create more risk.

### 36. Escalation ladder

Use the least invasive method first.

```text
1. lvu and state files
2. compile/install/MD5/activity logs
3. Moonbase source inspection
4. before/after snapshots
5. cache isolation
6. container reproduction
7. raw installwatch capture
8. source-level instrumentation
```

Stop when the question is answered.

### 37. Methodological discipline

A reliable advanced investigation follows:

```text
observe reality
→ preserve raw evidence
→ identify disagreement
→ formulate one hypothesis
→ instrument one boundary
→ execute one transition
→ compare
→ restore
→ document
```

### 38. Common mistakes

### Mistake 1: instrumenting production first

Use a disposable environment.

### Mistake 2: changing several variables

The result becomes ambiguous.

### Mistake 3: overwriting raw evidence

Interpretation should be separate.

### Mistake 4: forgetting to restore LSS code

Temporary instrumentation can become permanent accidental behavior.

### Mistake 5: trusting only state files

Check physical and runtime reality.

### Mistake 6: trusting only filesystem state

Ownership and policy may differ.

### Mistake 7: collecting everything without a question

More data is not automatically more knowledge.

### 39. Advanced-debugging checklist

Before:

```text
define one question
choose disposable environment
backup LSS code
preserve packages, depends, logs, module source
record environment and toolchain
```

During:

```text
change one variable
capture console
capture raw evidence if required
avoid unrelated operations
```

After:

```text
capture state again
diff before and after
verify runtime
restore instrumentation
restore module state
export evidence
write conclusion
```

### 40. Summary

Advanced LSS debugging is the controlled comparison of intent, events, ownership, state, and reality.

The full model is:

```text
Moonbase
→ declared intent

installwatch and logs
→ observed execution

manifest
→ interpreted ownership

packages and depends
→ persistent state

filesystem and runtime
→ actual result
```

The central rule is:

```text
instrument only the boundary that remains unclear
→ preserve evidence
→ restore the system

---

## 11. Module Removal and Configuration Preservation

### 1. Overview

Removing a module is not simply deleting a binary.

LSS removal may involve:

- loading the module's ownership manifest;
- checking package and dependency state;
- running remove hooks;
- protecting special paths;
- preserving modified configuration;
- deleting owned payload;
- removing empty exclusive directories;
- deleting Lunar-generated module artifacts;
- updating persistent state.

The practical model is:

```text
ownership
→ policy
→ filesystem change
→ persistent state change
```

### 2. The removal command

Use:

```bash
lrm module
```

Example:

```bash
lrm foremost
```

Typical success output:

```text
Removed module: foremost
```

Run as root.

### 3. Before removal

Always inspect:

```bash
lvu installed module
lvu depends module
lvu leert module
lvu where module
```

Also inspect raw state:

```bash
grep '^module:' /var/state/lunar/packages

grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Do not remove a module only because it is small or appears as a leaf.

### 4. Reverse dependency review

Use:

```bash
lvu depends module
lvu leert module
```

A result such as:

```text
foremost:
```

shows the reverse-tree root with no dependent branches.

A result listing modules means removal may affect them.

Check direct records:

```bash
grep -E '^[^:]+:module:'   /var/state/lunar/depends
```

### 5. Leaf status

List leaf modules:

```bash
lvu leafs | sort
```

Leaf means:

```text
no recorded reverse dependents
```

It does not mean:

```text
unused
non-critical
safe in every deployment
```

A leaf may still be:

- a boot tool;
- a network utility;
- a package-management component;
- a directly used administrator command;
- required by local scripts.

### 6. Inspect remove hooks

Locate the module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

Hooks can affect state outside the install manifest.

### 7. Preserve removal evidence

Module-specific logs may be deleted during removal.

Before `lrm`:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BACKUP=/root/lss-remove/$MODULE

mkdir -p "$BACKUP"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BACKUP/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BACKUP/md5sum-log"

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BACKUP/" 2>/dev/null || true

grep "^${MODULE}:" /var/state/lunar/packages   > "$BACKUP/package-record"

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BACKUP/depends-records" || true
```

### 8. Ownership manifest

The install manifest is:

```text
/var/log/lunar/install/module-version
```

It is the primary ownership source for removal.

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

It may contain:

- files;
- symlinks;
- shared directories;
- module-specific directories;
- documentation;
- manual pages;
- Lunar logs.

### 9. Ownership-driven removal

The normal model is:

```text
manifest path
→ classify
→ apply policy
→ remove when permitted
```

LSS does not reconstruct ownership only from the current filesystem.

The saved manifest is essential.

### 10. Regular files

Owned regular payload is removed.

Validated examples:

```text
/usr/bin/foremost
/usr/share/doc/foremost/CHANGES
/usr/share/doc/foremost/README
/usr/share/man/man8/foremost.8.gz
```

After removal:

```bash
test -e /usr/bin/foremost &&
  echo present ||
  echo missing
```

### 11. Shared directories

Shared directories are not blindly removed.

Validated retained paths:

```text
/usr
/usr/share
/usr/share/doc
```

These remained because they still contained content used by the rest of the system.

### 12. Exclusive directories

A module-specific directory may be removed after its contents disappear.

Validated example:

```text
/usr/share/doc/foremost
```

The practical behavior is:

```text
directory becomes empty
+ safe to remove
→ remove

directory remains shared or non-empty
→ retain
```

### 13. Symlinks

Owned symbolic links are part of the manifest and may be removed.

Inspect before:

```bash
while IFS= read -r path; do
  [ -L "$path" ] &&
    printf '%s -> %s
' "$path" "$(readlink "$path")"
done < install-log
```

A symlink may point to a file owned by the same module or by another module.

Ownership of the link and ownership of the target are separate.

### 14. Lunar-generated artifacts

Validated module-owned artifacts include:

```text
/var/log/lunar/install/module-version
/var/log/lunar/compile/module-version.xz
/var/log/lunar/md5sum/module-version
```

These may be deleted during `lrm`.

Therefore:

```text
preserve evidence before removal
```

is mandatory for forensic or documentation work.

### 15. Package-state update

Normal removal deletes the module record from:

```text
/var/state/lunar/packages
```

Validated transition:

```diff
-foremost:20260605:installed:1.5.7:8KB
```

No replacement `removed` record was created.

### 16. Dependency-state update

If the module has no dependency records, `/var/state/lunar/depends` may remain unchanged.

This was validated with `foremost`.

Modules participating in relationships may cause state changes.

Capture the full before/after diff:

```bash
cp -a /var/state/lunar/depends depends.before

lrm module

cp -a /var/state/lunar/depends depends.after

diff -u depends.before depends.after
```

### 17. Configuration under `/etc`

LSS treats `/etc` specially.

The MD5 log is used to compare current configuration with the installed version.

The validated normal behavior with `PRESERVE=on` is:

```text
unchanged configuration
→ remove

locally modified configuration
→ preserve
```

### 18. Unchanged configuration

In the first `foremost` removal test:

```text
/etc/foremost.conf
```

was unchanged.

Result:

```text
removed
```

Therefore:

```text
PRESERVE=on
≠ preserve every file under /etc
```

### 19. Modified configuration

After reinstalling, a controlled marker was appended to:

```text
/etc/foremost.conf
```

Then `lrm foremost` was run again.

Result:

```text
configuration remained
payload disappeared
package record disappeared
module logs disappeared
```

This validates local-change preservation.

### 20. Orphaned configuration

After package removal, a preserved configuration is:

```text
present on disk
not owned by an installed module
not backed by the removed module's current manifest
```

This is local orphaned state.

It may still be valuable.

It should not be mistaken for an installed package-owned file.

### 21. Handling orphaned configuration

Possible actions:

### Keep

Useful when reinstalling later or preserving local policy.

### Back up

```bash
cp -a /etc/module.conf   /root/module.conf.saved
```

### Compare with a new default

Reinstall in a controlled environment and compare.

### Remove deliberately

Only after confirming it is no longer needed.

### 22. `PRESERVE`

Inspect:

```bash
grep -n 'PRESERVE' /etc/lunar/config
```

Observed default-style setting:

```text
PRESERVE=${PRESERVE:-on}
```

The effective value may be influenced by environment or local configuration.

### 23. `PRESERVE=off`

Source analysis indicates a different branch for modified `/etc` files when preservation is disabled.

Expected behavior involves archival and removal.

This branch was not yet runtime-validated in the evidence series.

Treat it as source-derived until tested.

### 24. `PROTECTED`

`PROTECTED` paths may remain in ownership state but be blocked from deletion.

Conceptually:

```text
owned or tracked
+ protected policy
→ do not delete
```

Use protection for paths whose deletion would be unsafe.

Do not confuse protection with preservation of local modification.

### 25. `EXCLUDED`

`EXCLUDED` paths are filtered out of final ownership.

Conceptually:

```text
path touched during install
+ excluded policy
→ not claimed in final manifest
```

Because removal is manifest-driven, excluded paths are not normally removed through module ownership.

### 26. `PROTECTED` versus `EXCLUDED`

```text
PROTECTED
→ ownership may remain visible
→ deletion blocked

EXCLUDED
→ ownership not claimed
→ removal does not target it through the module manifest
```

These solve different problems.

### 27. Configuration versus protected path

A modified `/etc` file is preserved because checksum policy says local state changed.

A protected path remains because deletion policy forbids removal.

The observable result may be similar:

```text
file remains
```

But the reason is different.

Inspect both checksum and policy.

### 28. Remove hooks and configuration

A hook may modify or remove configuration independently of normal checksum policy.

Before removing a service or complex module, inspect:

```text
PRE_REMOVE
POST_REMOVE
```

Do not assume `/etc` behavior is controlled only by `PRESERVE`.

### 29. Upgrade removal

During rebuild, LSS may invoke:

```text
lrm --upgrade
```

Upgrade removal differs from ordinary user-requested removal.

Its purpose is to replace old ownership while preparing the new installation.

Do not use runtime results from normal `lrm` to assume every upgrade detail is identical.

### 30. Removal after failed installation

A failed install may leave:

- partial payload;
- incomplete manifest;
- stale package record;
- no package record;
- temporary installwatch state.

Before running `lrm`, inspect what ownership evidence exists.

If the manifest is incomplete, normal removal may not clean all partial files.

### 31. Missing manifest

If:

```text
package record exists
manifest missing
```

normal removal is unsafe because ownership evidence is incomplete.

Preferred recovery:

```text
preserve state
→ rebuild or reinstall same configuration
→ regenerate manifest
→ verify
→ remove if still desired
```

### 32. Manifest without package record

If:

```text
manifest exists
package record absent
```

possible causes include:

- interrupted removal;
- manual state edit;
- stale logs.

Preserve evidence.

Inspect activity log and filesystem before deciding whether the manifest is stale or payload remains.

### 33. Shared ownership

Search all manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Multiple matches indicate possible overlapping ownership.

Do not remove the path manually until the relationship is understood.

### 34. Files recreated after removal

A file may reappear because:

- a service recreates it;
- a login script regenerates it;
- another module owns it;
- a cache/index refresh recreates it;
- a hook runs after removal.

Compare timestamps and inspect hooks.

### 35. Safe removal test

For a low-risk module:

```text
select leaf
→ inspect role
→ inspect hooks
→ preserve logs and state
→ capture filesystem state
→ run lrm
→ compare state
→ verify payload
→ reinstall
→ verify clean restoration
```

This is the preferred method for studying removal behavior.

### 36. Filesystem-state capture

Before:

```bash
while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link	%s	%s
' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file	%s
' "$path"
  elif [ -d "$path" ]; then
    printf 'directory	%s
' "$path"
  else
    printf 'missing	%s
' "$path"
  fi
done < install-log > filesystem.before
```

Repeat after and compare:

```bash
diff -u filesystem.before filesystem.after
```

### 37. Reinstall after testing

After preserving evidence:

```bash
lin module
```

Verify:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

For configuration:

```bash
md5sum /etc/module.conf
```

Remove test markers or orphaned files before reinstalling when the goal is a pristine default.

### 38. Removal and services

Before removing a daemon:

- stop it;
- inspect active sockets;
- preserve configuration;
- inspect service-manager units;
- inspect hooks;
- verify dependent services.

After removal:

- confirm process is gone;
- confirm service entry behavior;
- confirm ports are closed;
- confirm configuration preservation.

### 39. Removal and plugins

A plugin-bearing module may extend LSS or another application.

Before removal:

- identify plugin paths;
- identify host process;
- inspect reload behavior;
- stop active operations;
- preserve plugin configuration.

A plugin file can disappear while the host process still has old code loaded.

Restart or reload as appropriate.

### 40. Removal and critical modules

Do not casually remove:

```text
lunar
glibc
gcc
bash
coreutils
filesystem components
init system
network base
boot components
shared toolchain libraries
```

Even if dependency state appears sparse.

Use a container or recovery environment.

### 41. Removal checklist

Before:

```text
confirm installed version
check reverse dependencies
inspect package record
inspect depends records
inspect remove hooks
save install and MD5 logs
save configuration
prepare rollback
```

During:

```text
capture console output
avoid unrelated operations
stop if unexpected warnings appear
```

After:

```text
verify package record is gone
verify dependency-state changes
verify payload
inspect preserved configuration
inspect shared directories
review activity log
```

### 42. Common mistakes

### Mistake 1: removing a dependency before rebuilding consumers

The current binaries may still need it.

### Mistake 2: assuming all `/etc` files remain

Unchanged configuration is normally removed.

### Mistake 3: assuming preserved configuration is still package-owned

It becomes orphaned local state.

### Mistake 4: deleting shared directories manually

They may contain files from other modules.

### Mistake 5: forgetting logs disappear

Preserve them first.

### Mistake 6: trusting leaf status alone

Operational use may exist outside LSS records.

### Mistake 7: ignoring remove hooks

They may have external side effects.

### 43. Removal model

```text
module record
→ ownership manifest
→ path classification
→ checksum and protection policy
→ payload removal
→ empty-directory cleanup
→ Lunar artifact removal
→ packages and depends update
→ preserved local state remains
```

### 44. Summary

LSS removal is a policy-controlled ownership transition.

The essential distinctions are:

```text
owned payload
→ normally removed

shared directory
→ retained when still needed

unchanged configuration
→ removed

modified configuration + PRESERVE=on
→ retained as local orphaned state

PROTECTED
→ tracked but deletion blocked

EXCLUDED
→ not claimed as ownership
```

The central rule is:

```text
inspect ownership and policy before removal
→ preserve evidence
→ verify both filesystem and persistent state afterward

---

## 12. Plugins and Extensibility

### 1. Overview

LSS can be extended through plugins.

Plugins allow additional behavior to be inserted without rewriting the core lifecycle.

They may:

- intercept an operation;
- provide an alternative implementation;
- add checks;
- add integration;
- modify decision flow;
- extend module behavior;
- reload dynamically in selected contexts.

This makes LSS extensible while preserving a relatively small core.

### 2. Plugin loading order

The runtime initialization order is conceptually:

```text
/etc/lunar/config
→ core function libraries
→ local configuration
→ path normalization
→ display and sound setup
→ plugin loading
```

This means plugins are loaded after the core functions exist.

A plugin can therefore extend or intercept known behavior rather than replacing an undefined environment.

### 3. Plugin directory

Plugins are loaded from the configured plugin directory.

A common conceptual location is:

```text
plugin.d
```

The exact path is determined by LSS configuration.

Inspect active configuration:

```bash
grep -R -n 'PLUGIN' /etc/lunar /var/lib/lunar/functions   2>/dev/null
```

List plugin files:

```bash
find /var/lib/lunar   -type f   -name '*.plugin'   -print
```

### 4. Module-installed plugins

A Moonbase module may install plugin files.

This means a normal software module can extend LSS itself.

Search Moonbase:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Search installed manifests:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/log/lunar/install
```

A plugin-bearing module deserves additional review before update or removal.

### 5. Registration order

Plugins are processed in registration order.

This matters because an earlier plugin may handle an operation and prevent later plugins from running.

Conceptually:

```text
plugin 1
→ plugin 2
→ plugin 3
→ default behavior
```

The chain may stop before reaching the end.

### 6. Return codes

Observed plugin return semantics are:

```text
0
→ operation handled successfully
→ stop processing

1
→ operation handled unsuccessfully
→ stop processing

2
→ continue to next plugin or default behavior
```

These codes are control-flow decisions, not only success/failure values.

### 7. Why return code 2 matters

In ordinary shell conventions:

```text
0
→ success

non-zero
→ failure
```

LSS plugin semantics add a third operational meaning:

```text
2
→ not handled here; continue
```

A plugin author must not treat every non-zero value as equivalent.

### 8. Plugin chain example

Conceptually:

```text
plugin A returns 2
→ plugin B runs

plugin B returns 0
→ chain stops successfully

default implementation
→ not called
```

Another case:

```text
plugin A returns 1
→ chain stops as failure
```

### 9. Plugins as interception points

A plugin may act at a specific lifecycle boundary.

Possible uses:

- source acquisition;
- cache recovery;
- dependency handling;
- policy checks;
- installation behavior;
- reporting;
- external integration.

The exact interception points depend on the plugin API exposed by LSS.

Inspect plugin code directly.

### 10. Inspecting a plugin

Read the full file:

```bash
sed -n '1,260p' /path/to/plugin.plugin
```

Search function definitions:

```bash
grep -n -E '^[[:space:]]*[a-zA-Z_][a-zA-Z0-9_]*[[:space:]]*\(\)'   /path/to/plugin.plugin
```

Search return values:

```bash
grep -n -E 'return[[:space:]]+[012]'   /path/to/plugin.plugin
```

Search registration:

```bash
grep -n -E 'register|plugin|hook'   /path/to/plugin.plugin
```

### 11. Plugin state and side effects

A plugin may interact with:

- filesystem;
- network;
- external services;
- package state;
- caches;
- logs;
- environment variables;
- module configuration.

Do not assume plugin effects appear in the module manifest.

A plugin can act outside the observed installation transaction.

### 12. Plugin provenance

Before trusting a plugin, determine:

```text
which module installed it?
which repository supplied it?
which version?
which manifest owns it?
which configuration enables it?
```

Search ownership:

```bash
grep -R -F -x '/path/to/plugin.plugin'   /var/log/lunar/install 2>/dev/null
```

### 13. Plugin removal

Removing the owning module may delete the plugin file.

However, a running `lin` process may already have loaded the plugin.

Therefore:

```text
file removed
≠ code unloaded from running process
```

Avoid removing plugin modules while LSS operations are active.

### 14. Runtime reload

Source analysis indicates plugin reload can be triggered through `USR1` into the parent `lin` process.

Conceptually:

```text
plugin set changes
→ signal parent lin
→ plugin environment reloads
```

This is advanced behavior.

Do not send signals to active package-manager processes without understanding the exact code path.

### 15. Inspecting active `lin` processes

```bash
ps -ef | grep '[l]in'
```

Inspect process tree:

```bash
pstree -ap | grep -A5 -B5 lin
```

Before plugin replacement or removal, wait for active operations to finish.

### 16. Plugin reload risks

Potential risks:

- partially changed plugin set;
- changed functions during active operation;
- mismatched state;
- plugin file removed before reload;
- new plugin loaded with old configuration;
- parent/child process disagreement.

Use reload only where explicitly supported.

### 17. Plugin debugging

Preserve:

```text
plugin file
registration order
LSS version
Moonbase revision
console output
activity log
environment
```

Run in a container.

Change one plugin at a time.

### 18. Detecting which plugin handled an operation

Possible methods:

- verbose console output;
- plugin-specific logs;
- activity log;
- temporary debug messages;
- controlled return-code changes;
- source-level tracing.

Use minimal instrumentation.

Do not modify several plugins simultaneously.

### 19. Plugin order conflicts

Two plugins may claim the same operation.

Because order matters:

```text
earlier plugin handles
→ later plugin never runs
```

A plugin can appear broken when it is simply shadowed.

Inspect registration sequence.

### 20. Default behavior fallback

If every plugin returns:

```text
2
```

the default LSS implementation may run.

This means a plugin can observe eligibility and decline handling without causing failure.

### 21. Successful interception

A plugin returning:

```text
0
```

takes responsibility for the operation.

It must leave the system in a state compatible with the expectations of the caller.

Success means more than command completion.

It may require:

- correct files;
- correct logs;
- correct state;
- correct return flow.

### 22. Failed interception

A return value of:

```text
1
```

halts the chain as failure.

Use this only when the plugin intentionally blocks or fails the operation.

A plugin that merely cannot handle the case should normally continue with:

```text
2
```

### 23. Plugin configuration

A plugin may use:

- global Lunar configuration;
- plugin-specific files;
- environment variables;
- module metadata;
- external credentials.

Search:

```bash
grep -R -n -E   '/etc/|CONFIG|config|credential|token|URL|PATH'   /path/to/plugin.plugin
```

Remove secrets before sharing debug bundles.

### 24. Plugin and cache behavior

A plugin may provide alternative cache or source behavior.

When a module installs unexpectedly quickly, determine whether:

- default cache resurrection ran;
- a plugin handled recovery;
- a plugin redirected source acquisition.

Inspect console and plugin order.

### 25. Plugin and dependency behavior

A plugin may influence dependency decisions.

This can change:

- selected provider;
- resolved relationship;
- build ordering;
- failure policy.

Compare `/var/state/lunar/depends` before and after plugin changes.

### 26. Plugin and ownership behavior

If a plugin installs files outside normal `prepare_install` observation, those paths may not appear in the module manifest.

This creates potential ownership gaps.

A plugin that changes installation behavior should preserve LSS ownership guarantees.

### 27. Safe plugin development

Recommended workflow:

```text
create plugin in isolated environment
→ register last in chain
→ return 2 by default
→ log decisions
→ handle one narrow case
→ test success
→ test decline
→ test failure
→ verify fallback
```

Start conservatively.

### 28. Safe plugin update

Before:

```bash
cp -a /path/to/plugin.plugin   /root/plugin.before

cp -a /var/state/lunar/packages   /root/packages.before

cp -a /var/state/lunar/depends   /root/depends.before
```

After:

- verify plugin syntax;
- verify registration;
- run a low-risk operation;
- inspect state;
- restore on unexpected behavior.

### 29. Safe plugin removal

Before removing the owning module:

1. identify active operations;
2. inspect plugin ownership;
3. inspect fallback behavior;
4. preserve plugin file;
5. stop or finish active `lin`;
6. remove module;
7. verify plugin directory;
8. run a low-risk LSS command.

### 30. Plugin syntax checking

Because plugins are shell code, use:

```bash
bash -n /path/to/plugin.plugin
```

This catches syntax errors, not semantic errors.

Also consider:

```bash
shellcheck /path/to/plugin.plugin
```

when available.

Review warnings manually.

### 31. Plugin logging

A useful plugin should log:

```text
interception point
decision
reason
return code
important external result
```

Avoid excessive logs.

The goal is traceability, not noise.

### 32. Plugin compatibility

A plugin may depend on:

- specific LSS function names;
- function arguments;
- global variables;
- directory layout;
- return semantics;
- lifecycle order.

An LSS update can therefore break a plugin even when the plugin file is unchanged.

Test plugins after core LSS updates.

### 33. Plugin API stability

Treat internal functions as unstable unless documented as public plugin interfaces.

A plugin built against undocumented globals may work but remain fragile.

Prefer small, explicit interfaces.

### 34. Extensibility boundaries

Plugins are appropriate when:

- behavior is optional;
- implementation is replaceable;
- core should remain small;
- external integration varies;
- fallback exists.

Plugins are less appropriate when:

- behavior is fundamental to package-state correctness;
- ordering cannot be controlled;
- ownership cannot be preserved;
- failures would corrupt core state.

### 35. Common mistakes

### Mistake 1: treating return 2 as failure

It means continue.

### Mistake 2: ignoring registration order

Earlier plugins may shadow later ones.

### Mistake 3: removing a plugin file during active `lin`

Loaded code may remain in memory.

### Mistake 4: installing files outside ownership tracking

This creates unmanaged state.

### Mistake 5: modifying several plugins at once

Failure attribution becomes difficult.

### Mistake 6: relying on undocumented core globals

Compatibility becomes fragile.

### Mistake 7: returning 0 without completing expected state

The caller assumes successful handling.

### 36. Plugin review template

```bash
PLUGIN=/path/to/plugin.plugin

echo '=== SYNTAX ==='
bash -n "$PLUGIN"

echo
echo '=== FUNCTIONS ==='
grep -n -E   '^[[:space:]]*[a-zA-Z_][a-zA-Z0-9_]*[[:space:]]*\(\)'   "$PLUGIN"

echo
echo '=== RETURNS ==='
grep -n -E 'return[[:space:]]+[012]' "$PLUGIN"

echo
echo '=== REGISTRATION ==='
grep -n -E 'register|hook|plugin' "$PLUGIN"

echo
echo '=== EXTERNAL PATHS ==='
grep -n -E '/etc/|/var/|/usr/|https?://' "$PLUGIN"
```

### 37. Plugin troubleshooting model

```text
operation requested
→ registration order
→ plugin eligibility
→ return code
→ next plugin or fallback
→ persistent state
→ runtime result
```

Debug each transition.

### 38. Extensibility and system intent

Plugins should extend system intent, not obscure it.

A good plugin makes clear:

- what it handles;
- when it declines;
- what state it changes;
- what fallback remains;
- how it is removed.

### 39. Summary

LSS plugins provide ordered, replaceable interception.

The central semantics are:

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ not handled; continue
```

The operational rule is:

```text
understand registration order
→ preserve ownership and state guarantees
→ test fallback
→ avoid changing plugins during active operations

---

## 13. Policy States: Held, Exiled, and Enforced Modules

### 1. Overview

LSS stores more than installation state.

The package-state database can also preserve policy decisions.

Known state terms include:

```text
installed
held
exiled
enforced
```

These states describe different administrative intentions.

The central distinction is:

```text
installation state
→ what is present

policy state
→ what LSS should or should not do
```

### 2. Package-state database

The database is:

```text
/var/state/lunar/packages
```

Observed record format:

```text
module:date:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

The third field may represent one or more states.

Do not treat it as a simple yes/no installed flag.

### 3. Why policy states exist

A source-based rolling system needs more than automatic progression.

Administrators may need to:

- keep a known-good version;
- prevent an unwanted module from returning;
- require a module to remain present;
- protect a transition in progress;
- preserve local system policy.

Policy states make that intent persistent.

### 4. `installed`

`installed` means the module is recorded as present.

It normally corresponds to:

- package record;
- installed version;
- size;
- install manifest;
- MD5 log;
- payload on disk.

However, persistent state and filesystem reality can drift.

Always verify critical cases.

### 5. `held`

A held module is intentionally prevented from normal update progression.

Conceptually:

```text
module remains installed
→ current version is retained
→ normal update should not replace it
```

Typical reasons:

- known regression in newer version;
- compatibility requirement;
- local patch;
- ABI freeze;
- production stability;
- pending validation.

### 6. When to hold a module

Good reasons:

```text
new version breaks required functionality
toolchain transition incomplete
dependent software not ready
local patch exists only for current version
critical service needs validation first
```

Bad reason:

```text
avoid understanding the update indefinitely
```

A hold should be documented and reviewed.

### 7. Risks of long-term holds

A held module may accumulate:

- security exposure;
- dependency incompatibility;
- ABI divergence;
- stale configuration;
- unsupported patches;
- upgrade complexity.

Record:

```text
why held
since when
which version
which dependents
what condition ends the hold
```

### 8. Inspecting held modules

Search:

```bash
awk -F: '$3 ~ /(^|\+)held(\+|$)/ {print}'   /var/state/lunar/packages
```

For one module:

```bash
grep '^module:' /var/state/lunar/packages
```

Inspect reverse dependents before changing policy:

```bash
lvu depends module
lvu leert module
```

### 9. `exiled`

An exiled module is intentionally excluded from the system's accepted package universe.

Conceptually:

```text
module should not be installed
→ dependency resolution should respect that policy
```

Possible reasons:

- incompatible software;
- unwanted provider;
- security policy;
- replaced implementation;
- licensing or operational policy;
- broken module.

### 10. Exiled versus uninstalled

```text
uninstalled
→ not currently present

exiled
→ explicitly forbidden or rejected
```

This distinction matters.

An uninstalled dependency may be installed later.

An exiled dependency represents a negative administrative decision.

### 11. Exiled versus disabled optional dependency

A disabled optional dependency means:

```text
this module does not use that feature
```

An exiled module means:

```text
this module should not be accepted on the system
```

The scope differs.

Optional disablement belongs to one module relationship.

Exile is broader system policy.

### 12. When to exile a module

Possible uses:

- prevent a conflicting provider;
- prevent an obsolete implementation;
- enforce local security policy;
- block accidental installation;
- preserve a deliberate alternative stack.

Review conflicts and dependents first.

### 13. Risks of exile

Exiling a module may make other modules impossible to install.

Potential results:

- dependency resolution failure;
- feature loss;
- provider conflict;
- incomplete system update.

Before exile:

```bash
lvu depends module
lvu leert module
grep -E '^[^:]+:module:' /var/state/lunar/depends
```

### 14. `enforced`

An enforced module is required by policy to remain part of the system.

Conceptually:

```text
module must be present
→ system policy should restore or retain it
```

Possible uses:

- essential administration tool;
- mandatory security component;
- required system service;
- organizational baseline;
- critical provider.

### 15. Enforced versus required dependency

A required dependency belongs to another module's dependency graph.

An enforced module belongs to system policy.

```text
required dependency
→ needed by module A

enforced module
→ required by administrator or system policy
```

A module can be enforced even when no package depends on it.

### 16. Enforced versus installed

```text
installed
→ current presence

enforced
→ persistent requirement for presence
```

An installed module can be removed.

An enforced module should not be casually removed because policy expects it to remain.

### 17. Combined state lists

The state field may encode multiple terms.

Conceptually:

```text
installed+held
installed+enforced
```

Do not assume exact combination syntax without inspecting real records.

Parse the state field as a list, not a single fixed value.

### 18. Inspecting state combinations

```bash
awk -F: '{
  printf "%-30s %-24s %-16s %s\n",
         $1, $3, $4, $5
}' /var/state/lunar/packages
```

Search a term:

```bash
awk -F: '$3 ~ /held/ {print}'   /var/state/lunar/packages
```

Preserve the full original record.

### 19. Policy state and update behavior

Before update:

```bash
grep '^module:' /var/state/lunar/packages
```

If held:

```text
do not assume normal update will proceed
```

If enforced:

```text
do not assume removal is administratively valid
```

If exiled:

```text
do not assume dependency installation is allowed
```

### 20. Policy state and rebuild

A rebuild may or may not preserve policy terms depending on the operation.

Before:

```bash
grep '^module:' /var/state/lunar/packages   > package.before
```

After:

```bash
grep '^module:' /var/state/lunar/packages   > package.after

diff -u package.before package.after
```

Verify both version and state-list.

### 21. Policy state and removal

Before `lrm`, inspect:

```bash
grep '^module:' /var/state/lunar/packages
```

Do not remove an enforced module without first changing the policy intentionally.

A held module may still be removable, but the hold explains administrative intent.

An exiled module may already be absent.

### 22. Policy state and dependency resolution

Possible interactions:

```text
dependency needed
+ dependency exiled
→ resolution conflict

module enforced
+ dependency missing
→ restoration or failure path

module held
+ dependent requires newer ABI
→ update conflict
```

Policy and dependency state must be considered together.

### 23. Policy review before system update

List policy states:

```bash
awk -F: '$3 != "installed" {print}'   /var/state/lunar/packages
```

Review each record before a large update.

Create a policy report:

```bash
awk -F: '{
  if ($3 ~ /held|exiled|enforced/)
    printf "%s:%s:%s:%s:%s\n", $1, $2, $3, $4, $5
}' /var/state/lunar/packages   > /root/lunar-policy-report
```

### 24. Documenting policy decisions

For each policy state, record:

```text
module
state
reason
date
owner
expected review date
exit condition
related modules
```

Persistent state without rationale becomes technical debt.

### 25. Temporary holds

A temporary hold should have a clear exit condition.

Examples:

```text
remove hold after LLVM family rebuild passes
remove hold after service regression is fixed
remove hold after dependent module supports new ABI
```

Avoid indefinite anonymous holds.

### 26. Provider policy

Policy states are useful for alternative providers.

Example:

```text
provider A enforced
provider B exiled
```

This makes system intent explicit.

Before switching:

```text
remove old policy
→ configure replacement
→ rebuild dependents
→ verify runtime
→ apply new policy
```

### 27. Policy drift

Policy drift occurs when:

```text
state says held
but module was manually replaced

state says enforced
but payload is missing

state says exiled
but module is present

state says installed
but manifest is missing
```

Detect through cross-checking.

### 28. Policy consistency checks

For enforced modules:

```bash
awk -F: '$3 ~ /enforced/ {print $1}'   /var/state/lunar/packages |
while read -r module; do
  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "missing record: $module"
done
```

Also verify manifests and payload.

For exiled modules, inspect whether they are unexpectedly installed.

### 29. Do not edit the database manually

Manual editing may:

- break state-list syntax;
- desynchronize related state;
- hide policy history;
- create unsupported combinations.

Use supported LSS commands.

If a repair is unavoidable:

```text
preserve original database
→ document reason
→ make minimal change
→ verify all affected state
```

### 30. Policy state backup

Before policy changes:

```bash
cp -a /var/state/lunar/packages   /root/packages.before-policy-change
```

Capture relevant dependency state:

```bash
cp -a /var/state/lunar/depends   /root/depends.before-policy-change
```

Afterward, diff both.

### 31. Policy and recovery

A recovery should restore not only payload but policy.

Example:

```text
restore module files
but lose held state
→ system may update unexpectedly later
```

Back up:

```text
packages
depends
Moonbase revision
local policy notes
```

### 32. Policy and automation

Automation should respect administrator policy.

A safe automated updater must:

- skip held modules;
- reject exiled modules;
- preserve enforced modules;
- report conflicts;
- avoid silently changing policy.

Automation should not reinterpret intent without evidence.

### 33. Common mistakes

### Mistake 1: treating held as installed

Held adds update policy.

### Mistake 2: treating exiled as merely absent

It is an explicit rejection.

### Mistake 3: treating enforced as a dependency

It is system-level policy.

### Mistake 4: forgetting policy during rollback

Payload and state must both be restored.

### Mistake 5: leaving holds undocumented

Future administrators cannot evaluate them.

### Mistake 6: forcing dependency resolution around exile

Resolve the policy conflict first.

### 34. Policy review template

```bash
echo '=== SPECIAL POLICY STATES ==='

awk -F: '
  $3 ~ /held|exiled|enforced/ {
    printf "module=%s date=%s states=%s version=%s size=%s\n",
           $1, $2, $3, $4, $5
  }
' /var/state/lunar/packages
```

For each module:

```bash
MODULE=name

grep "^${MODULE}:" /var/state/lunar/packages
lvu depends "$MODULE"
lvu leert "$MODULE"
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

### 35. Safe policy-change model

```text
inspect current state
→ identify reason
→ inspect dependencies
→ preserve databases
→ change through LSS
→ verify record
→ test intended behavior
→ document decision
```

### 36. Policy hierarchy

A useful conceptual model:

```text
administrator intent
→ held / exiled / enforced

module intent
→ required / optional dependencies

installed result
→ package record and manifest

runtime reality
→ actual files and behavior
```

Conflicts between layers must be resolved explicitly.

### 37. Summary

LSS policy states preserve administrator intent across normal package operations.

```text
held
→ keep current version

exiled
→ reject module presence

enforced
→ require module presence
```

These states are distinct from ordinary installation and dependency state.

The central rule is:

```text
inspect policy before update or removal
→ change it deliberately
→ preserve the rationale

---

## 14. Building and Testing a Local Module

### 1. Overview

A local Moonbase module is useful for:

- private software;
- experimental updates;
- unreleased patches;
- hardware-specific tools;
- local services;
- testing packaging changes;
- preparing upstream contributions.

A safe local-module workflow is:

```text
create
→ inspect
→ build in isolation
→ verify ownership
→ test runtime
→ remove
→ verify cleanup
→ reinstall
```

Do not begin with a critical package.

### 2. Use an isolated environment

Preferred environments:

- rootless Podman container;
- chroot;
- disposable VM;
- test machine;
- snapshot-capable system.

The test environment should allow:

```text
root access
Moonbase access
normal lin/lrm/lvu use
evidence export
easy rollback
```

### 3. Choose a simple first module

A good first module:

- builds quickly;
- has few or no dependencies;
- installs one small executable;
- has no service;
- has no plugin;
- has no complex `/etc` state;
- has no reverse dependents.

Avoid:

```text
glibc
gcc
bash
lunar
kernel
init system
network base
shared toolchain components
```

### 4. Local section

Keep local modules separate from upstream-managed sections when possible.

Example layout:

```text
/var/lib/lunar/moonbase/local/utils/hello-lunar
```

This provides:

- clear provenance;
- easier backup;
- fewer merge conflicts;
- safer Moonbase updates.

Follow the conventions of the active Moonbase.

### 5. Minimal module structure

A minimal module directory:

```text
hello-lunar/
├── DETAILS
└── BUILD
```

More complex modules may add:

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

Start with the minimum.

### 6. `DETAILS`

A minimal conceptual `DETAILS` contains:

```bash
MODULE=hello-lunar
VERSION=1.0
SOURCE=$MODULE-$VERSION.tar.gz
SOURCE_URL=https://example.org/releases/
SOURCE_VFY=sha256:...
WEB_SITE=https://example.org/
ENTERED=20260717
UPDATED=20260717
SHORT="Small example program"

cat << EOF
A small example program used to demonstrate a local Lunar module.
EOF
```

Use the actual syntax and verification conventions accepted by the active Moonbase.

### 7. Source verification

Never omit source verification casually.

Verify:

```text
source file name
version
download URL
checksum or signature
upstream identity
```

Calculate SHA-256:

```bash
sha256sum hello-lunar-1.0.tar.gz
```

Record the result according to Moonbase conventions.

### 8. `BUILD`

For a conventional project:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

For a simple Makefile project:

```bash
make &&
prepare_install &&
make install PREFIX=/usr
```

The critical boundary is:

```text
prepare_install
```

Call it before system installation.

### 9. Why `prepare_install` matters

Before `prepare_install`:

```text
build tree activity
```

After `prepare_install`:

```text
filesystem installation observed by installwatch
```

Installing to `/usr` before this boundary may produce incomplete ownership tracking.

### 10. Using default helpers

When appropriate:

```bash
default_build
```

or:

```bash
default_make
```

These reduce boilerplate.

Still inspect their behavior in the current LSS version.

Do not assume a helper matches every upstream build system.

### 11. Compiler flags

Use the Lunar build environment rather than hardcoding arbitrary optimization.

A module may adjust:

```bash
export CFLAGS+=" -fcommon"
```

only when a compatibility reason exists.

Document unusual flags.

For toolchain-specific software, select compiler family explicitly when necessary.

### 12. Dependencies

Add `DEPENDS` only for real relationships.

Separate:

```text
required
optional
build-time
runtime
provider choice
```

Do not add dependencies simply because they happen to be installed on the development system.

Test in a clean environment to discover hidden assumptions.

### 13. Optional dependencies

Optional dependencies should map clearly to functionality.

The user should understand:

```text
enable dependency
→ gain feature

disable dependency
→ omit feature
```

Pass explicit positive and negative build flags when possible.

Avoid accidental upstream auto-detection.

### 14. Configuration and options

Use `CONFIGURE` or `OPTIONS` when the user must choose behavior.

Good configuration questions are:

- stable;
- meaningful;
- reproducible;
- directly connected to build output.

Avoid exposing every upstream switch without operational value.

### 15. Patches

Place patches in the module directory.

Apply them in `PRE_BUILD` or `BUILD` according to conventions.

A patch should have:

```text
purpose
upstream status
affected version
author/source
removal condition
```

Test that it still applies after a version update.

### 16. Shell quality

Moonbase modules are executable shell.

Use:

- clear `&&` chains;
- quoted variables where appropriate;
- explicit failure behavior;
- minimal global side effects;
- standard helpers;
- readable comments for unusual logic.

Validate syntax:

```bash
bash -n BUILD
bash -n PRE_BUILD
```

### 17. Pre-build inspection

Before running `lin`:

```bash
MODULE=hello-lunar
SECTION=local/utils
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

find "$MODULE_DIR" -maxdepth 2 -type f -print | sort

sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

Inspect all hooks and dependencies.

### 18. Record baseline

```bash
BASE=/root/lss-local-module/$MODULE

mkdir -p "$BASE/before" "$BASE/after-install" "$BASE/after-remove"

cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

cp -a "$MODULE_DIR"   "$BASE/before/module-definition"

env | sort > "$BASE/before/environment"
```

Record Moonbase revision and local diff.

### 19. Build and install

Run:

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/install-console.log"
```

Do not run unrelated package operations during the test.

### 20. Verify package state

```bash
lvu installed "$MODULE"

grep "^${MODULE}:"   /var/state/lunar/packages
```

Expected record:

```text
module:date:installed:version:size
```

Verify dependency records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends || true
```

### 21. Verify generated artifacts

```bash
VERSION=$(lvu installed "$MODULE")

ls -l   "/var/log/lunar/install/${MODULE}-${VERSION}"   "/var/log/lunar/md5sum/${MODULE}-${VERSION}"

ls -l   /var/log/lunar/compile/${MODULE}-${VERSION}.*   2>/dev/null || true
```

If `ARCHIVE=on`:

```bash
ls -l /var/cache/lunar/${MODULE}-${VERSION}-*
```

### 22. Inspect the manifest

```bash
cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

Check that it contains:

- intended files;
- intended symlinks;
- expected directories;
- no build-tree paths;
- no accidental host files;
- no unwanted static libraries;
- no unrelated configuration.

### 23. Classify paths

```bash
MANIFEST="/var/log/lunar/install/${MODULE}-${VERSION}"

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link	%s	%s
' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file	%s
' "$path"
  elif [ -d "$path" ]; then
    printf 'directory	%s
' "$path"
  else
    printf 'missing	%s
' "$path"
  fi
done < "$MANIFEST"
```

A missing final path requires investigation.

### 24. Verify runtime

For a command:

```bash
command -v hello-lunar
hello-lunar --version
hello-lunar
```

For a library:

```bash
readelf -d /usr/lib/libhello.so
pkgconf --modversion hello
```

For a service:

- inspect service file;
- start in test environment;
- inspect process and logs;
- run a functional request.

### 25. Verify linking

```bash
ldd /usr/bin/hello-lunar
```

Look for:

```text
not found
```

Compare runtime linkage with declared dependencies.

### 26. Verify installed size

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "$MANIFEST"

echo "${SIZE}KB"
```

Compare with `/var/state/lunar/packages`.

### 27. Rebuild test

Run:

```bash
cp -a "$MANIFEST"   "$BASE/after-install/install.before-rebuild"

lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"

cp -a "$MANIFEST"   "$BASE/after-install/install.after-rebuild"
```

Compare:

```bash
diff -u   "$BASE/after-install/install.before-rebuild"   "$BASE/after-install/install.after-rebuild"
```

A simple deterministic module should normally produce stable ownership.

### 28. Removal test

Before removal, preserve logs:

```bash
cp -a "$MANIFEST"   "$BASE/after-install/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/after-install/md5sum-log"
```

Run:

```bash
lrm "$MODULE" 2>&1 |
  tee "$BASE/remove-console.log"
```

### 29. Verify removal

Package record:

```bash
grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "package record removed"
```

Verify payload paths from saved manifest.

Shared directories should remain.

Module-specific empty directories should disappear.

### 30. Reinstall test

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/reinstall-console.log"
```

Verify:

```bash
lvu installed "$MODULE"
grep "^${MODULE}:" /var/state/lunar/packages
```

Run the functional test again.

### 31. Configuration preservation test

Only for a non-critical test configuration.

Sequence:

```text
install
→ preserve original checksum
→ modify config
→ lrm
→ verify modified config remains
→ remove or archive orphan
→ reinstall
→ verify clean default
```

Do not perform this test on production configuration.

### 32. Dependency test

For a module with dependencies:

```text
install dependency
→ install module
→ inspect depends state
→ inspect reverse tree
→ disable optional feature
→ rebuild
→ inspect relationship change
```

Use one optional relationship at a time.

### 33. Cache test

When safe:

```text
build normally
→ verify cache
→ remove module
→ reinstall with cache available
→ observe whether cache is used
→ verify manifest and runtime
```

Then isolate cache and compare with fresh build.

### 34. Failure-path test

A mature module should also be tested under controlled failure.

Examples:

- unavailable source;
- wrong checksum;
- missing required dependency;
- intentionally failing patch;
- invalid compiler.

Verify:

- clear failure phase;
- useful compile log;
- no false package record;
- no unmanaged partial payload;
- recoverable next run.

### 35. Hidden dependency detection

Test in a minimal container.

A module that builds only on the developer's full system may depend on undeclared tools or libraries.

Search errors for:

```text
command not found
header missing
library missing
pkg-config missing
```

Add real dependencies rather than relying on ambient software.

### 36. Install path review

Look for unwanted paths:

```text
/usr/local
/opt unexpected
home directory
build directory
temporary directory
host-specific path
```

Normalize paths in the module where appropriate.

### 37. `/etc` review

Configuration should be:

- intentional;
- documented;
- tracked by MD5;
- compatible with `PRESERVE`;
- free of generated secrets;
- not overwritten blindly.

Do not ship machine-specific credentials.

### 38. Service integration

For a service module, include the files appropriate to the target init system.

Test:

```text
install
→ service definition present
→ start
→ stop
→ restart
→ remove
→ service definition gone
→ preserved config behavior correct
```

Do not start services unexpectedly during package build unless policy explicitly requires it.

### 39. Local module evidence bundle

Preserve:

```text
module definition
source checksum
Moonbase revision
environment
toolchain
install console
compile log
manifest
MD5 log
package record
dependency records
runtime test
remove console
before/after diffs
```

Archive:

```bash
tar -cJf "${MODULE}-test-evidence.tar.xz"   "$BASE"
```

### 40. Preparing for upstream contribution

Before proposing a module:

- remove local-only assumptions;
- use canonical source;
- verify license;
- verify checksums;
- minimize dependencies;
- make optional features explicit;
- test clean install;
- test rebuild;
- test removal;
- document unusual choices;
- follow Moonbase style.

### 41. Review questions

Ask:

```text
Does the module describe real dependencies?
Does prepare_install occur before system installation?
Is final ownership clean?
Are optional features explicit?
Does removal clean payload safely?
Are modified configs preserved correctly?
Can the module rebuild reproducibly?
Does it work in a minimal environment?
```

### 42. Common mistakes

### Mistake 1: testing only successful compile

Install, runtime, removal, and reinstall also matter.

### Mistake 2: building on a full host

Hidden dependencies remain undetected.

### Mistake 3: installing before `prepare_install`

Ownership may be incomplete.

### Mistake 4: hardcoding local paths

The module becomes non-portable.

### Mistake 5: enabling every optional feature

This weakens administrator choice.

### Mistake 6: omitting removal test

Manifest errors remain hidden.

### Mistake 7: editing state databases manually

Use normal LSS operations.

### 43. Minimal acceptance criteria

A local module is ready for wider testing when:

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
no local test modifications remain
```

### 44. Summary

Building a local module is a complete lifecycle exercise.

```text
definition
→ dependency resolution
→ build
→ observed installation
→ ownership
→ runtime
→ removal
→ restoration
```

The central rule is:

```text
test the module as LSS will live with it
→ not only as a source tree that compiles

---

## 15. System Recovery and State Repair

### 1. Overview

LSS maintains several layers of system state:

```text
filesystem payload
install manifest
MD5 log
package-state database
dependency-state database
Moonbase definition
cache
activity history
```

A healthy system keeps these layers aligned.

Recovery is needed when one or more layers disagree.

The first rule is:

```text
preserve evidence before repair
```

### 2. Common inconsistency classes

Typical cases:

```text
package record exists, payload missing
package record exists, manifest missing
manifest exists, package record missing
dependency record references missing module
payload exists, no ownership record
configuration survives without package ownership
cache exists, installed state absent
Moonbase definition changed, installed state old
```

Each case requires a different response.

### 3. Do not repair by reflex

Avoid immediate actions such as:

```text
editing packages by hand
editing depends by hand
deleting stale-looking logs
manually copying binaries into /usr
manually unpacking cache into /
```

These actions may hide the real inconsistency.

First determine which layer is authoritative for the question.

### 4. Build a recovery bundle

Before repair:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-recovery/$MODULE

mkdir -p "$BASE"
```

Capture state:

```bash
cp -a /var/state/lunar/packages   "$BASE/packages.before"

cp -a /var/state/lunar/depends   "$BASE/depends.before"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package-record.before" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/depends-records.before" || true
```

Capture logs:

```bash
cp -a /var/log/lunar/install/${MODULE}-*   "$BASE/" 2>/dev/null || true

cp -a /var/log/lunar/md5sum/${MODULE}-*   "$BASE/" 2>/dev/null || true

cp -a /var/log/lunar/compile/${MODULE}-*   "$BASE/" 2>/dev/null || true
```

Capture Moonbase definition:

```bash
SECTION=$(lvu where "$MODULE" 2>/dev/null)
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

test -d "$MODULE_DIR" &&
  cp -a "$MODULE_DIR" "$BASE/module-definition"
```

### 5. Record physical state

If a manifest exists:

```bash
MANIFEST=$(ls /var/log/lunar/install/${MODULE}-* 2>/dev/null | head -1)
```

Classify paths:

```bash
if [ -n "$MANIFEST" ]; then
  while IFS= read -r path; do
    if [ -L "$path" ]; then
      printf 'link	%s	%s
' "$path" "$(readlink "$path")"
    elif [ -f "$path" ]; then
      printf 'file	%s
' "$path"
    elif [ -d "$path" ]; then
      printf 'directory	%s
' "$path"
    else
      printf 'missing	%s
' "$path"
    fi
  done < "$MANIFEST"     > "$BASE/filesystem-state.before"
fi
```

### 6. Package record exists, payload missing

Condition:

```text
packages
→ says installed

filesystem
→ important payload missing
```

Possible causes:

- manual deletion;
- filesystem damage;
- interrupted operation;
- restored state without payload.

Preferred recovery:

```text
preserve evidence
→ rebuild or reinstall module
→ regenerate payload
→ verify manifest
→ verify runtime
```

Do not delete the package record simply to remove the warning.

### 7. Package record exists, manifest missing

Condition:

```text
package state
→ present

ownership evidence
→ absent
```

Risk:

```text
lrm cannot reliably know what to remove
```

Preferred recovery:

```text
rebuild or reinstall the same configuration
→ regenerate manifest and MD5 log
→ compare payload
→ verify package state
```

Avoid removal before ownership is restored.

### 8. Manifest exists, package record missing

Condition:

```text
ownership log
→ present

package record
→ absent
```

Possible causes:

- interrupted removal;
- manual state edit;
- stale log;
- partial recovery.

Investigate:

```bash
grep "$MODULE" /var/log/lunar/activity
```

Check payload paths.

Do not assume the manifest still represents current ownership.

### 9. Payload exists, no manifest and no package record

Condition:

```text
files exist
→ no LSS ownership
```

Possible causes:

- manual install;
- preserved config;
- interrupted package lifecycle;
- external tool;
- orphaned files.

Identify provenance before assigning ownership.

Search all manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

### 10. Dependency record references missing module

Condition:

```text
depends
→ relationship exists

packages
→ dependency absent
```

Possible causes:

- interrupted removal;
- manual state edit;
- failed dependency installation;
- stale configuration.

Preferred action:

```text
inspect dependent module
→ inspect DEPENDS/CONFIGURE/OPTIONS
→ reconfigure or rebuild dependent
→ verify new dependency state
```

Do not simply delete the dependency record unless no supported path exists.

### 11. Installed dependency with no reverse dependents

Condition:

```text
module installed
→ no current reverse relationships
```

Possible interpretations:

- direct user-installed tool;
- orphaned dependency;
- provider retained intentionally;
- stale dependency state.

Use:

```bash
lvu leafs
lvu depends module
lvu leert module
```

Review role before removal.

### 12. Preserved orphaned configuration

Condition:

```text
/etc file remains
package removed
```

This may be correct behavior under:

```text
PRESERVE=on
```

Check saved MD5 evidence.

Actions:

- keep;
- archive;
- compare with new default;
- remove deliberately.

Do not force ownership onto a new module without review.

### 13. Cache exists, package absent

This is normal.

```text
cache
→ reusable payload

package state
→ current installation
```

A cache alone does not mean installed.

Use LSS to reinstall.

Avoid manual extraction into `/`.

### 14. Package installed, cache absent

Also normal.

The module can be installed without a reusable cache.

Possible reasons:

- `ARCHIVE=off`;
- cache deleted;
- cache creation failed;
- older installation.

Ownership and removal still depend on manifest and state.

### 15. Stale package size

A recorded size may differ from current calculation.

Reproduce:

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "$MANIFEST"

echo "${SIZE}KB"
```

If manifest and payload are healthy, a rebuild may refresh size metadata.

### 16. Stale Moonbase versus installed state

Moonbase may change while installed state remains old.

This is not necessarily corruption.

Compare:

```bash
lvu installed module
lvu version module
```

Inspect Moonbase history.

Rebuild only after reviewing changed dependencies and configuration.

### 17. Interrupted rebuild

Possible states:

```text
old package removed
partial new payload
no final manifest
stale package record
old cache available
```

Recovery sequence:

```text
stop further updates
→ preserve all evidence
→ inspect activity and compile log
→ inspect package record
→ inspect old cache
→ reinstall known-good version/configuration
→ verify
```

### 18. Interrupted removal

Possible states:

```text
package record removed
some payload remains
logs partially removed
configuration preserved
```

Use saved or surviving manifest.

If no manifest remains, compare against cache, backup, or module reinstall in a test environment.

### 19. Repair through rebuild

A rebuild is often the safest repair when:

- module definition still exists;
- source is available;
- configuration is known;
- toolchain works;
- payload is incomplete;
- manifest is missing or stale.

Use:

```bash
lin -c module
```

Preserve console output.

### 20. Repair through reinstall

A reinstall is appropriate when:

- module is no longer recorded as installed;
- payload is absent or incomplete;
- cache or source is trusted;
- clean ownership must be recreated.

Use:

```bash
lin module
```

Then verify package state, manifest, MD5 log, and runtime.

### 21. Repair through cache

Use cache when:

- source build is unavailable;
- trusted archive exists;
- configuration matches;
- target triplet matches;
- provenance is known.

Restore through LSS.

Do not unpack manually unless performing expert emergency recovery.

### 22. Repair through rollback

Rollback sources:

```text
trusted old cache
filesystem snapshot
container backup
system backup
old Moonbase revision
old source archive
```

A complete rollback restores:

```text
payload
manifest
MD5 log
package state
dependency state
configuration
```

### 23. Manual database repair

Manual repair is last resort.

Before:

```bash
cp -a /var/state/lunar/packages   /root/packages.pre-manual-repair

cp -a /var/state/lunar/depends   /root/depends.pre-manual-repair
```

Document:

- exact inconsistency;
- why supported operations failed;
- exact line changed;
- expected result;
- validation performed.

Make the smallest possible edit.

### 24. Manual manifest reconstruction

Also last resort.

Use only when:

- no rebuild possible;
- no cache;
- no backup;
- payload must be removed or recovered;
- ownership can be established from strong evidence.

Possible evidence:

- package cache content;
- upstream install list;
- filesystem timestamps;
- activity logs;
- old backups;
- another identical system.

A guessed manifest is dangerous.

### 25. State consistency matrix

### Healthy installed module

```text
package record present
manifest present
MD5 log present
payload present
dependency state coherent
```

### Healthy removed module

```text
package record absent
manifest absent
payload absent
preserved local config possible
```

### Suspicious state

```text
record present, manifest absent
record absent, manifest present
active dependency, missing module
manifest paths missing
payload present, no ownership
```

### 26. Verify after repair

Package state:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

Ownership:

```bash
cat /var/log/lunar/install/module-version
```

Dependency state:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Runtime:

```bash
command -v program
ldd /usr/bin/program
program --version
```

Configuration:

```bash
md5sum /etc/module.conf
```

### 27. System-wide state audit

List package records:

```bash
wc -l /var/state/lunar/packages
```

List manifests:

```bash
find /var/log/lunar/install   -maxdepth 1   -type f   -printf '%f
' |
  sort
```

Look for obvious mismatches between installed records and manifests.

Automated audit should report, not auto-repair.

### 28. Check installed modules without manifests

Conceptual audit:

```bash
awk -F: '$3 ~ /installed/ {print $1 ":" $4}'   /var/state/lunar/packages |
while IFS=: read -r module version; do
  test -f "/var/log/lunar/install/${module}-${version}" ||
    echo "missing manifest: ${module}-${version}"
done
```

Review results manually.

Some historical exceptions may exist.

### 29. Check manifests without records

```bash
for manifest in /var/log/lunar/install/*; do
  name=$(basename "$manifest")
  module=${name%-*}

  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "manifest without obvious record: $name"
done
```

Version parsing by filename can be ambiguous when names contain dashes.

Treat output as a lead, not proof.

### 30. Check missing manifest paths

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < manifest
```

Some paths may be intentionally absent after later transitions.

Investigate context.

### 31. Check dependency references

```bash
awk -F: '{print $1; print $2}'   /var/state/lunar/depends |
sort -u |
while read -r module; do
  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "dependency-state module absent: $module"
done
```

This may report exiled or special-state modules.

Interpret policy state.

### 32. Repair and policy states

Do not lose:

```text
held
exiled
enforced
```

during recovery.

Restoring only `installed` may change administrator intent.

Preserve full records.

### 33. Repair and local Moonbase changes

If the installed module came from a local patch:

```text
rebuild from upstream current module
→ may not reproduce the installed result
```

Preserve active module definition and local diff before recovery.

### 34. Repair and optional features

A reinstall with different optional choices may restore a working module but change functionality.

Preserve dependency records and configuration answers.

Recovery should restore intended behavior, not merely a successful binary.

### 35. Repair and toolchain drift

A module rebuilt with a new compiler may differ from the damaged installation.

Record:

```text
CC
CXX
CFLAGS
CXXFLAGS
LDFLAGS
toolchain versions
```

Use a trusted cache if exact rebuild is impossible.

### 36. Recovery checkpoints

After each major repair step:

```bash
cp -a /var/state/lunar/packages   "$BASE/packages.checkpoint-N"

cp -a /var/state/lunar/depends   "$BASE/depends.checkpoint-N"
```

Record what changed.

Do not perform several major repairs without intermediate checkpoints.

### 37. Recovery report

A useful report includes:

```text
initial inconsistency
evidence preserved
probable cause
chosen repair path
commands executed
state diffs
runtime verification
remaining uncertainty
```

This turns recovery into reusable knowledge.

### 38. Common mistakes

### Mistake 1: deleting state to match missing files

This hides damage.

### Mistake 2: restoring files without ownership

Future removal and updates remain broken.

### Mistake 3: restoring ownership without payload

State remains false.

### Mistake 4: ignoring dependency state

Consumers may still be inconsistent.

### Mistake 5: losing policy states

Administrator intent changes silently.

### Mistake 6: rebuilding with different options unknowingly

Functionality may drift.

### Mistake 7: repairing before preserving evidence

Cause becomes harder to understand.

### 39. Recovery decision model

```text
identify inconsistent layers
→ preserve all relevant evidence
→ choose supported repair path
→ restore alignment
→ verify persistent state
→ verify filesystem
→ verify runtime
→ document
```

### 40. Summary

LSS recovery is the restoration of agreement between several state layers.

```text
Moonbase
→ intended module behavior

packages and depends
→ persistent administrative state

manifest and MD5
→ ownership and installed checksums

filesystem
→ physical payload

runtime
→ actual functionality
```

The central rule is:

```text
repair alignment
→ not only the visible symptom

---

## 16. Reference Commands and File Locations

### 1. Core commands

### Install a module

```bash
lin module
```

Example:

```bash
lin foremost
```

### Rebuild an installed module

```bash
lin -c module
```

Example:

```bash
lin -c xxhash
```

### Remove a module

```bash
lrm module
```

Example:

```bash
lrm foremost
```

### Inspect a module

```bash
lvu installed module
lvu version module
lvu where module
lvu depends module
lvu leert module
```

### 2. Essential command meanings

```text
lin module
→ install module

lin -c module
→ rebuild module through upgrade-style replacement

lrm module
→ remove module through its ownership manifest

lvu installed module
→ show installed version

lvu version module
→ show version available in active Moonbase

lvu where module
→ show Moonbase section

lvu depends module
→ show installed reverse dependents

lvu leert module
→ show reverse dependency tree
```

### 3. Locate a module in Moonbase

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s
' "$MODULE_DIR"
```

List files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

### 4. Common module files

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
CONFLICTS
```

Inspect:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

### 5. Package-state database

Location:

```text
/var/state/lunar/packages
```

Observed format:

```text
module:date:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

Lookup:

```bash
grep '^module:' /var/state/lunar/packages
```

List installed modules:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {print $1}'   /var/state/lunar/packages | sort
```

### 6. Policy states

Known state terms:

```text
installed
held
exiled
enforced
```

Meaning:

```text
installed
→ currently recorded as present

held
→ retain current version

exiled
→ reject module presence

enforced
→ require module presence
```

Inspect special states:

```bash
awk -F: '$3 ~ /held|exiled|enforced/ {print}'   /var/state/lunar/packages
```

### 7. Dependency-state database

Location:

```text
/var/state/lunar/depends
```

Observed format:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

Inspect forward relationships:

```bash
grep '^module:' /var/state/lunar/depends
```

Inspect reverse relationships:

```bash
grep -E '^[^:]+:module:' /var/state/lunar/depends
```

Inspect both:

```bash
MODULE=name

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

### 8. Leaf modules

List:

```bash
lvu leafs | sort
```

Leaf means:

```text
no recorded reverse dependents
```

It does not automatically mean safe or unused.

### 9. Install manifests

Location:

```text
/var/log/lunar/install
```

Per-module file:

```text
/var/log/lunar/install/module-version
```

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

Count paths:

```bash
wc -l /var/log/lunar/install/module-version
```

### 10. Manifest path classification

```bash
MANIFEST=/var/log/lunar/install/module-version

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link	%s	%s
' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file	%s
' "$path"
  elif [ -d "$path" ]; then
    printf 'directory	%s
' "$path"
  else
    printf 'missing	%s
' "$path"
  fi
done < "$MANIFEST"
```

### 11. MD5 logs

Location:

```text
/var/log/lunar/md5sum
```

Per-module file:

```text
/var/log/lunar/md5sum/module-version
```

Inspect:

```bash
cat /var/log/lunar/md5sum/module-version
```

Compare a configuration file:

```bash
md5sum /etc/module.conf

grep '/etc/module.conf'   /var/log/lunar/md5sum/module-version
```

### 12. Compile logs

Location:

```text
/var/log/lunar/compile
```

Typical file:

```text
module-version.xz
```

Read:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search errors:

```bash
xzgrep -n -i   -E 'error|failed|fatal|undefined|not found|cannot'   /var/log/lunar/compile/module-version.xz
```

### 13. Activity log

Location:

```text
/var/log/lunar/activity
```

Search module history:

```bash
grep 'module' /var/log/lunar/activity
```

Follow live:

```bash
tail -f /var/log/lunar/activity
```

### 14. Cache directory

Location:

```text
/var/cache/lunar
```

Find a module cache:

```bash
ls -lh /var/cache/lunar/module-*
```

Typical form:

```text
module-version-target-triplet.tar.xz
```

Inspect archive:

```bash
tar -tJf archive.tar.xz
```

Verify XZ integrity:

```bash
xz -t archive.tar.xz
```

### 15. Global configuration

Main file:

```text
/etc/lunar/config
```

Common settings:

```text
ARCHIVE
PRESERVE
REAP
```

Inspect:

```bash
grep -n -E 'ARCHIVE|PRESERVE|REAP'   /etc/lunar/config
```

### 16. Important LSS paths

```text
/sbin/lin
/sbin/lrm
/bin/lvu

/etc/lunar
/var/lib/lunar/functions
/var/lib/lunar/moonbase
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/activity
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
/var/cache/lunar
/usr/lib/installwatch.so
```

### 17. Environment inspection

```bash
uname -a
id

env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS|PATH|PKG_CONFIG_PATH)='
```

Toolchain:

```bash
gcc --version
clang --version
ld --version
make --version
```

### 18. Installed versus Moonbase version

```bash
MODULE=name

printf 'installed: %s
'   "$(lvu installed "$MODULE" 2>/dev/null)"

printf 'moonbase:  %s
'   "$(lvu version "$MODULE" 2>/dev/null)"
```

### 19. Reverse dependency review

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -E "^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Run before removal or provider replacement.

### 20. Safe evidence capture

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-evidence/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

### 21. Capture console output

Install:

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/install-console.log"
```

Rebuild:

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"
```

Remove:

```bash
lrm "$MODULE" 2>&1 |
  tee "$BASE/remove-console.log"
```

### 22. Compare state before and after

```bash
cp -a /var/state/lunar/packages   "$BASE/after/packages"

cp -a /var/state/lunar/depends   "$BASE/after/depends"

diff -u   "$BASE/before/packages"   "$BASE/after/packages"

diff -u   "$BASE/before/depends"   "$BASE/after/depends"
```

### 23. Compare manifests

```bash
diff -u install.before install.after
```

No output means byte-for-byte identity.

### 24. Reproduce installed size

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < /var/log/lunar/install/module-version

echo "${SIZE}KB"
```

Use `du -k` explicitly.

Do not recursively measure manifest directories.

### 25. Shared ownership search

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Multiple results require review.

### 26. Missing path check

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < /var/log/lunar/install/module-version
```

### 27. Binary and linkage checks

```bash
command -v program
file /usr/bin/program
ldd /usr/bin/program
```

Missing libraries:

```bash
ldd /usr/bin/program | grep 'not found'
```

Library metadata:

```bash
readelf -d /usr/lib/libexample.so
```

### 28. Pkg-config checks

```bash
pkgconf --exists package
pkgconf --modversion package
pkgconf --print-requires package
pkgconf --print-requires-private package
```

### 29. Moonbase Git inspection

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase branch --show-current
git -C /var/lib/lunar/moonbase rev-parse HEAD
git -C /var/lib/lunar/moonbase diff
```

Save local changes:

```bash
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

### 30. Compare module revisions

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   section/module
```

History:

```bash
git -C /var/lib/lunar/moonbase log --   section/module
```

### 31. Plugin inspection

Find plugins:

```bash
find /var/lib/lunar   -type f   -name '*.plugin'   -print
```

Search Moonbase plugin installers:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Syntax check:

```bash
bash -n /path/to/plugin.plugin
```

Return semantics:

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ continue
```

### 32. Remove-hook inspection

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

### 33. Build-hook inspection

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

### 34. Optional-feature inspection

```bash
for file in DEPENDS CONFIGURE OPTIONS BUILD; do
  if [ -f "$MODULE_DIR/$file" ]; then
    echo "=== $file ==="
    sed -n '1,240p' "$MODULE_DIR/$file"
  fi
done
```

### 35. Configuration preservation checks

Inspect effective setting:

```bash
grep -n 'PRESERVE' /etc/lunar/config
```

Check current config checksum:

```bash
md5sum /etc/module.conf
```

Compare with saved MD5 log.

Expected with `PRESERVE=on`:

```text
unchanged /etc file
→ removed

modified /etc file
→ preserved
```

### 36. Cache backup

```bash
mkdir -p /root/lunar-cache-backup

cp -a /var/cache/lunar/module-version-*   /root/lunar-cache-backup/

sha256sum /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/SHA256SUMS
```

### 37. Evidence export from container

```bash
podman cp   lunar-dev:/root/lss-evidence/.   ~/LSS-Evidence/
```

Verify:

```bash
find ~/LSS-Evidence -type f | sort
```

### 38. Quick install verification

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
test -f /var/log/lunar/install/module-version
```

### 39. Quick rebuild verification

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
diff -u install.before install.after
```

### 40. Quick removal verification

```bash
grep '^module:' /var/state/lunar/packages || true

test -e /known/payload/path &&
  echo present ||
  echo missing
```

### 41. Quick dependency safety check

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

### 42. Quick troubleshooting sequence

```text
1. Read console output.
2. Identify lifecycle phase.
3. Open compile log.
4. Find first meaningful error.
5. Inspect module BUILD and hooks.
6. Inspect toolchain environment.
7. Inspect dependency state.
8. Compare before and after.
9. Test runtime.
```

### 43. Quick recovery sequence

```text
preserve evidence
→ identify inconsistent layers
→ rebuild or reinstall through LSS
→ regenerate manifest and state
→ verify filesystem
→ verify runtime
```

### 44. Critical distinctions

```text
Moonbase
→ intended module behavior

installwatch
→ raw observed filesystem events

install manifest
→ final ownership

MD5 log
→ installed checksums

packages
→ installation and policy state

depends
→ relationship state

cache
→ reusable installation payload

activity log
→ historical transitions

runtime
→ actual behavior
```

### 45. Operational rules

```text
inspect before changing
preserve logs before removal
rebuild before removing an optional dependency
do not edit state databases casually
do not treat leaf as automatically safe
do not treat cache as complete truth
do not treat same version as same build
verify runtime after successful installation
```

### 46. Minimal administrator checklist

Before install or rebuild:

```text
inspect module
inspect dependencies
record current state
verify source and toolchain
```

Before removal:

```text
inspect reverse dependents
inspect hooks
save manifest and MD5 log
check /etc state
```

After any change:

```text
verify packages
verify depends
verify manifest
verify filesystem
verify runtime
```

### 47. Final summary

LSS administration follows one consistent discipline:

```text
understand intent
→ inspect state
→ preserve evidence
→ perform one controlled operation
→ verify ownership
→ verify persistent state
→ verify runtime
```

This reference is the compact operational map for that discipline.

---

## About this guide

This publication draft consolidates the first complete LSS User Guide. It is intended as the source document for adaptation and integration into the Lunar Linux website.

The guide documents the observable behavior of the current LSS architecture and emphasizes safe administration, evidence preservation, explicit state inspection, and controlled recovery.
