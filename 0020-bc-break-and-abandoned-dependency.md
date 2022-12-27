# 20. BC Break & PHP Abandoned Dependency

Date: 2022-11-16

## Status

In discussion

## Context

As said in the ADR #17, impact on production package in Composer dependencies are limited :
- Updates are only possible only if the package has not a strong impact
- Removal is not allowed.

Packages can be discontinued (with or without an alternative) in a patch or minor release. These packages can therefore be a possible source of security holes.

## Decision

1. Allow in a minor version to remove a discontinued package

## Consequences

It is possible to have a removal of a package in minor version but only in the case of a discontinued package.