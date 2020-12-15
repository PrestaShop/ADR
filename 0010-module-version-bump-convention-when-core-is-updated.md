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

I suggest to apply the following rule:

WHEN a new module version requires to bump the PrestaShop minimum version,

IF the bump is patch (ex: PS 1.7.6.2 => 1.7.6.4)
THEN the module version must be bumped to at least the next minor version

IF the bump is minor or major
THEN the module version must be bumped to at least the next major version

<hr>

That would mean, for module A who bumps PS minimum version from PS 1.7.6 to PS 1.7.7, the new version must be a major version.

## Alternative solution

An alternative solution is to be a little more flexible:

WHEN a new module version requires to bump the PrestaShop minimum version,

IF the bump is patch (ex: PS 1.7.6.2 => 1.7.6.4)
THEN the module version must be bumped to at least the next patch version

IF the bump is minor
THEN the module version must be bumped to at least the next minor version

IF the bump is major
THEN the module version must be bumped to at least the next major version

<hr>

That would mean, for module A who bumps PS minimum version from PS 1.7.6 to PS 1.7.7, the new version must be a minor version.