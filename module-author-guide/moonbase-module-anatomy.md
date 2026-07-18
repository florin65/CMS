---
title: "Moonbase Module Anatomy"
description: "The purpose of common files in a Moonbase module."
---

# Moonbase Module Anatomy

Common files include:

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

## DETAILS

Defines source identity, version, source URLs, checksums, descriptions, and metadata.

Version alone is not complete build identity. Options, providers, compiler, patches, and local Moonbase changes also matter.

## BUILD

Defines the build and installation procedure.

Review:

- command ordering;
- configure and build flags;
- destination paths;
- patches and generated files;
- cleanup behavior;
- whether installation begins only after `prepare_install`.

## DEPENDS

Defines required and optional relationships. Optional dependencies may also encode build choices.

## CONFIGURE and OPTIONS

These establish module-specific choices and switches passed to the underlying build system.

## Hooks

Lifecycle hooks may update services, indexes, generated state, boot metadata, or other system-wide resources. Treat them as executable package policy.
