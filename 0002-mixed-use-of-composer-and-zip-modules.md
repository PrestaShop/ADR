# 2. Mixed use of composer and zip modules

Date: 2019-02-15

## Status

Accepted

## Context

The normal behavior of a module is to be self-contained, meaning that its code and all its dependencies are stored in the module's directory, including `vendor` directory and autoloader.

When developing and building PrestaShop, native modules aren't downloaded from marketplace, they are required using composer.

In difference with the normal behavior described above, a module installed using composer will have its autoloader and dependencies merged into the core's and placed in the core's `vendor` directory.

This can prove problematic for native modules:

- When the module is uninstalled, both its dependencies and autoloader are left behind in the core.
- If the module is upgraded using the marketplace sources, the dependencies are now available twice: once in the core and once in the module.

Alternate systems to avoid having modules leak code into the core have been proved unpractical:

- Including modules via [Composer script handler](https://github.com/PrestaShop/composer-script-handler) or git clone is too slow.
- Using submodules would be a step backwards.
- Retrieving modules from the marketplace would be slow as well.

## Decision

1. [The module managment system must be changed to be fully based on composer](https://github.com/PrestaShop/PrestaShop/issues/12586). This will require major changes in the marketplace and will have to be analyzed for feasibility.
2. In the meantime, we will keep using composer to include native modules.
3. Native modules must prepend their autoloader.
4. To avoid leaving dependencies in the core, no composer dependencies are to be added to native modules until step 1 has been resolved.

## Consequences

- A module installed via composer and uninstalled will leave its autoloader data behind. It's not a problem as far as we can see.
- A module installed via composer and upgraded will have old autoloader data in the core, but it won't be an issue as long as Decision No. 3 is respected.
- Because of Decision No. 4, we may face technical difficulties in the future.
