# 4. Registration of a bundle-like module 

Date: 2020-02-13

## Status

In discussion

## Context

The normal behavior of a module is to be self-contained, meaning that its code and all its dependencies are stored in the module's directory.

As we are migrating to Symfony, the necessity to use some techniques like DependencyInjection, EventListener, ... come to us.

To avoid scanning the modules directory everytime to know if such services exists within some of the existing modules, the idea came to consider a module like a bundle.

Some documentations about bundles

https://symfony.com/doc/3.4/bundles.html

https://symfony.com/doc/3.4/bundles/best_practices.html

Bundles are registered when the Kernel boots and each bundle can register its own services, routing, controllers, templates, ... and this is automated by Symfony when the structure conventions are respected.

When registering a bundle, you have to instantiate the main class which extends `Symfony\Component\HttpKernel\Bundle\Bundle` and add it to the returned array of method `registerBundles` of the AppKernel. And that's it.

But, how can we determinate the fully qualified class name of a Module's bundle when:

* Bundle class can be theorically in any sub-directory of the module.
* There are no conventions about what a module namespace should look like.

Other aspects have to be taken in consideration:

* PrestaShop will support bundle-like and non bundle-like modules.
* A bundle may need to include third-party bundles (?)
* A module can be present in the filesystem but not be activated.

## Decision

1. Each module which have to be treated like a bundle must have a `manifest.php` file at the root of it with the list of bundles to load.
2. The bundles to load must be in an array and instantiated.
3. If one of the bundles doesn't extends `Symfony\Component\HttpKernel\Bundle\Bundle` the Manifest will be ignored.
4. The module's bundles will be aggregated into the file `_PROJECT_DIR_/config/module_bundles.php`.

##### Structure of Manifest.php
```php
<?php

return [
    'bundles' => [
        new \MyCompanyName\Module\CoffeMakerBundle(),
        new \ThirdPartyCompany\AdditionalBundle(),
    ],
];

```

##### Structure of generated module_bundles.php
```php
<?php
return [
	'module_wake_up' => [
		new MyCompanyName\Module\CoffeMakerBundle(),
		new ThirdPartyCompany\AdditionalBundle(),
	],
	'ps_other_module' => [
		new Prestashop\Module\OtherModuleBundle(),
	],
];
```

Indexing the generated array by the module name allow us to don't load bundles of disabled modules.

## Points of friction

The module developer must be careful about the third-party bundles he use. If a component is already used by the Prestashop Core, the module one will be used (because it is loaded last) and that can lead to unpredicted behaviours. (@TODO test this assert).


## Votes by maintainers

|                   | Yes | NO |
|-------------------|-----|----|
| @Progi1984        |     |    |
| @sowbiba          | X   |    |
| @jolelievre       |     |    |
| @matthieu-rolland |     |    |
| @matks            |     |    |
| @PierreRambaud    |     |    |
| @eternoendless    |     |    |
| @atomiix          |     |    |
| @NeOMaking        |     |    |
