---
title: "Validated Case Studies"
description: "Compact examples showing how LSS events, ownership, removal, and configuration preservation behave."
---

# Validated Case Studies

These examples illustrate concrete LSS behavior.

## `xxhash`: transient installation activity

A controlled rebuild captured raw installwatch activity around:

```text
/usr/lib/libxxhash.a
```

The raw stream included:

```text
open
chmod
unlink
```

The file was therefore created or opened, had permissions applied, and was later removed during the same lifecycle.

It did not appear in the final install manifest.

This demonstrates:

```text
raw event history
≠ final ownership
```

The final manifest before and after the rebuild was identical.

## `xxhash`: installed size

The recorded installed size matched a manual sum of regular files listed in the manifest.

Observed rules:

- regular files counted;
- directories excluded;
- symlinks excluded;
- Lunar-generated logs listed in the manifest contributed;
- explicit `du -k` was required to avoid an interactive `du -h` alias.

## `foremost`: normal removal

Before removal, the manifest included:

- `/etc/foremost.conf`;
- the executable;
- documentation;
- a manual page;
- shared directories;
- Lunar compile, install, and MD5 logs.

Normal removal produced:

```text
package record removed
payload removed
unchanged `/etc/foremost.conf` removed
module-specific empty documentation directory removed
shared `/usr` and `/usr/share` directories retained
module-specific Lunar logs removed
```

## `foremost`: modified configuration

After reinstall, `/etc/foremost.conf` was modified locally.

With `PRESERVE=on`, removal produced:

```text
payload removed
package record removed
module logs removed
modified configuration retained
```

The retained file was local orphaned configuration rather than ownership of an installed module.

## Operational lessons

```text
final manifest
→ final ownership, not complete event history

PRESERVE=on
→ preserves modified `/etc` state, not every `/etc` file

shared directories
→ not removed blindly

module logs
→ preserve them before removal when they are needed as evidence
```
