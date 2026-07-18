---
title: "State and Return Semantics"
description: "Compact reference for LSS package state, dependency records, path policy, and plugin return values."
---

# State and Return Semantics

## Package record

Observed form:

```text
module:YYYYMMDD:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

The state field is a list, not merely a Boolean installed flag.

Known terms:

```text
installed
→ module is recorded as present

held
→ retain current version

exiled
→ reject module presence

enforced
→ require module presence
```

Possible combinations should be interpreted from the complete state list.

## Dependency record

Observed form:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

Inspect forward and reverse records:

```bash
grep '^module:' /var/state/lunar/depends

grep -E '^[^:]+:module:' \
  /var/state/lunar/depends
```

## Path policy

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

The `PRESERVE=off` modified-file branch should be verified against the active LSS implementation before relying on its exact archival behavior.

## Plugin returns

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ not handled; continue
```

Registration order affects which plugin receives control.

## Ownership semantics

```text
raw installwatch event
→ a filesystem event occurred

final install manifest entry
→ LSS attributes final ownership
```

Raw events may include temporary paths that do not survive into final ownership.

## Installed-size semantics

Validated behavior:

```text
regular files in final manifest
→ counted

directories
→ not counted

symlinks
→ not counted

Lunar-generated regular log files in manifest
→ may contribute
```

Use `du -k` explicitly when reproducing the size.
