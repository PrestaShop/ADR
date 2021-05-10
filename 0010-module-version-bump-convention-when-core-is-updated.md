# 10. Module version bump convention when Core is updated

Date: 2020-12-15

## Status

In discussion

## Context

When working on PrestaShop open source module, we release new versions.

Sometimes a new version of a module requires to bump the required minimum PrestaShop Core versions, for example
- module A v3.4.2 can run with PrestaShop 1.7.1 to 1.7.5
- module A v4.0.0 can run with PrestaShop 1.7.4 to 1.7.7

This is because the new version of the module A required some code that was introduced in PS 1.7.4 .

It is important to understand that **bumping required minimum PrestaShop Core version means dropping some compatibility** for the module. The module previous version could run with some PS versions, the new version cannot.

[Example with ps_searchbar](https://github.com/PrestaShop/ps_searchbar/pull/24)

**The topic of this ADR is "what version bump should we apply when PrestaShop minimum version is bumped?"**

### SemVer opinion

[SemVer](https://semver.org/) states that dropping a dependency support is not a BC break. So according to SemVer, we can do whatever we want.

This opinion is the topic a long debate, [many people disagree with it](https://github.com/semver/semver/issues/148). But this is what SemVer says today.

As for me, I side with the people who disagree with it :smile: . See below my point of view.

### Module user opinion

As a merchant, I think it would feel very weird to allow bumping PrestaShop minimum version in patch versions.

For example, let's say I use PrestaShop 1.7.6 with module A v3.2.5 .

I want to upgrade module A to v3.2.6 which was released yesterday. And I can't ! Because module A v3.2.6 requires PrestaShop 1.7.7 .

_I think many merchants would be surprised to see we accept that a patch version for module A requires an upgrade from the Core._

I believe that, beyond SemVer, people attach "meaning" to patch/minor/major versions bump. They expect patch versions to have some continuity and to not require a different environment.

## Decision

I have 3 candidates for a rule, the 3rd one was suggested by @PierreRambaud .

Note that I consider
- a PrestaShop core bump 1.7.6.2 => 1.7.6.4 is a patch bump
- a PrestaShop core bump 1.7.6.2 => 1.7.7.0 is a minor bump
- a PrestaShop core bump 1.7.6.2 => 1.8.0.0 is a major bump

I ignore the "1." in front of PrestaShop versions.

### Candidate 1 : bump to major when Core minimum version is bumped to minor

WHEN a new module version requires to bump the PrestaShop minimum version,

IF the core bump is a patch bump (ex: PS 1.7.6.2 => 1.7.6.4)
THEN the module version must be bumped to at least the next minor version (ex: ps_searchbar 2.0.1 => 2.2.0)

IF the bump is minor or major (ex: PS 1.7.6.2 => 1.7.7.0)
THEN the module version must be bumped to at least the next major version (ex: ps_searchbar 2.0.1 => 3.0.0)

### Candidate 2 : bump to minor when Core minimum version is bumped to minor

WHEN a new module version requires to bump the PrestaShop minimum version,

IF the core bump is a patch bump (ex: PS 1.7.6.2 => 1.7.6.4)
THEN the module version must be bumped to at least the next patch version (ex: ps_searchbar 2.0.1 => 2.0.2)

IF the bump is minor (ex: PS 1.7.6.2 => 1.7.7.0)
THEN the module version must be bumped to at least the next minor version (ex: ps_searchbar 2.0.1 => 2.1.0)

IF the bump is major (ex: PS 1.7.6.2 => 1.8.0.0)
THEN the module version must be bumped to at least the next major version (ex: ps_searchbar 2.0.1 => 3.0.0)

### Candidate 3 : bump to major when Core minimum version is bumped to minor AND stop using SemVer for modules

Suggested by @PierreRambaud : this candidate is identical to the 1st candidate but also suggests to stop following SemVer for module versions.

SemVer is good because it helps users understand whether the project introduces Breaking Changes or not. But we are not sure PrestaShop community extends our modules, and backward compatibility is interesting for modules only if the module is extended or reused.

@PierreRambaud suggests to stop using SemVer for modules and modify upon release the version number according to the type of changes introduced: bug fixes (patch), or new features (minor), or important changes (major).
