# 20. Module versioning convention when minimum compatibilty with the core is updated

Date: 2023-02-21

## Status

Approved

## Context

When working on PrestaShop open source module, we release new versions.

Sometimes a new version of a module requires to bump the required minimum PrestaShop Core versions, for example
- `module_a` can run with PrestaShop 1.7.1 to 1.7.8
- later, with a new version of `module_a`, it can run with minimum PrestaShop 8.0

To this day, it is not clear whether these two scenarios require upgrading major or minor versions of the module.

**The topic of this ADR is "what version bump should we apply when PrestaShop Core minimum version is bumped?"**

It is important to understand that **bumping required minimum PrestaShop Core version means dropping some compatibility** for the module. The module's previous version could run with some PS versions, but the new version cannot. It can be considered as a BC Break, this is why we should agree on bumping major version of the module.

[Example with ps_searchbar](https://github.com/PrestaShop/ps_searchbar/pull/24)

### Consequences

Every time we change minimum compatible version of PrestaShop Core for a module, we need to bump module major version. Example:

IF we change minimum compatible PrestaShop Core v ersion (ex: PS 1.7.6.2 => 1.7.7.0, PS 1.7.4 -> 8.0)
THEN the module version must be bumped to the next major version (ex: ps_searchbar 2.0.1 => 3.0.0).
