# 7. Module advanced installation

Date: 2020-03-09

## Status

In discussion

## Context

As of today (v1.7.7) PrestaShop allows a few advanced features for Symfony developers, among which:
- automatic require of `vendor/autoload.php`
- possibility to define services for front AND for admin using [Symfony container](https://symfony.com/doc/current/service_container.html)
- possibility to use [Doctrine](https://symfony.com/doc/current/doctrine.html) entities

However there is a limitation to all these features they are managed based on the **installed modules** list which is built at the kernel booting.

As a consequence when installing a new module all these features are unavailable resulting in a few drawbacks:
- you can't use your module namespaces (you can still require the autoload file "manually" in the module class file)
- Doctrine entities not available (you must use ObjectModel or DB to init your tables and fixtures)
- services not available (you can self construct your services to use them)

Although workaround and alternatives exist, this is not acceptable as we want to offer developers a better development environment and experience.

## Autoload and Doctrine solution

These are the two easier to deal with, a solution has already been proposed in this PR [PrestaShop/PrestaShop#17706](https://github.com/PrestaShop/PrestaShop/pull/17706) which could be split for those two issues.
It also includes an helper class/service that would allow to easily update the database schema based on Doctrine.

## Container solutions

A few solutions have been imagined and discussed, here are the three main possibilities along with their advantages and drawbacks.

### Dynamically integrate new services in current container

This was the initial proposal, already implemented in this PR https://github.com/PrestaShop/PrestaShop/pull/17706
The idea is to:
- build a container on runtime
- this container only scans for module's services.yml
- it is able to fetch the required dependencies from the kernel's container
- finally it injects the new services from the module inside the kernel's container

**Advantages:**
- dynamic on runtime
- low memory consumption
- single request/process

**Drawbacks:**
- not compatible with compiler pass (for now)
- not compatible with kernel's extensions (probably)
- not compatible with kernel's or module's bundles (coming feature)
- custom method that seems to work in a defined scope but could prove its limitation and force heavy development/adjustments to make it work fully in the future

### Perform an installation in two times

Since the core problem is that the installing module is not in the installed modules list, the idea is to split the installation process.
- first the module is installed as they have always been in the first request/process
- now that the module is installed and its dependencies are loaded, a second request/process performs the `Module:postInstall` step where the module can use all the available features to finish its installation

**Advantages:**
- simple solution
- safe as we are sure all the module's stack will be available on the second call

**Drawbacks:**
- developers need to manage a new postInstall interface in their modules
- requires two process/requests
- error management needs to be handled in case the post install fails, and uninstall the module
- CLI process require to launch a sub process for the post install
- this will be a long process as the container needs to be cleaned (and so rebuilt) twice

### The end point hack

To be able to change "dynamically" the modules list before the kernel is booted we could use special request parameters (or CLI arguments) to prepare the modules list in advance, and inject the installing module in it.
In order for the kernel to correctly rebuild the container the whole process must be performed in "dev" environment so that the cached container is not used directly.

**Advantages:**
- single request/process
- the kernel is correctly booted with the required module's list
- optimized memory use as only one kernel is needed

**Drawbacks:**
- it's a big hack
- it requires to manage every single end point before the kernel is booted (unless it is performed in the kernel class)
- maybe this can open potential security issues

### Container reboot solution

One solution that was suggested at the end of the meeting was to use the native kernel reboot function
https://github.com/symfony/symfony/blob/bdd66aafc0ac254b47f21f5c8fbdfbffb378a469/src/Symfony/Component/HttpKernel/Kernel.php#L148

This needs to be testes but this method seems to be able to provoke the kernel shutdown and reboot, so we could update the modules list at this moment so that all the installing feature are taken into account on reboot.

**Advantages:**
- clean way provided by the Symfony framework
- the whole kernel is rebuilt so all bundles/extensions should be correctly initialised
- single process/request
- maybe we could improve performances (for now the whole cache is cleaned after a module action, maybe the reboot is able to rebuild the cache more proficiently)

**Drawbacks:**
- this needs to be tested
- as PrestaShop manages the container poorly we need to be careful about services that would be cached statically (the most obvious being `SymfonyContainer::getInstance` but there are probably other cases)
- what happens to previously fetched services that would already be "stored"
- memory consumption (double kernel, double memory use)
- inform module developers that they mustn't call services early (in the module constructor) but when they really use them (good practice anyway)

## Consequences

We need to test the alternative solutions to see if they meet all our needs and choose the most efficient.
