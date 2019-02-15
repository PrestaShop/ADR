# 3. Doctrine for modules

Date: 2019-02-15

## Status

Accepted

## Context

It is not possible to use doctrine today in Modules, some module developers requested this possibility. Either to use Doctrine entities or for Doctrine query builder.

Defining entites is one but using the doctrine is another. For admin modules used in symfony pages it is rather easy to get back the service (although it's a bit of a trick you need to use `PrestaShop\PrestaShop\Adapter\SymfonyContainer::getInstance`) which is not the cleaner way to do it. But you can't access to this service in legacy controllers (neither admin nor front ones).

## Decision

### Defining entites

The idea would to allow modules to have a configuration for their entites, doctrine has multiple ways to declare its entites we need to choose which one we allow:
- Annotations (do we force to use a specific folder `src/Entity` by convention)
- YAML config file (in config/doctrine.yml or config/doctrine/*.orm.yml?)
- XML config file (in config/doctrine.xml or config/doctrine/*.orm.xml?)

### Enable access to doctrine service

Symfony context is already manage, the problem is for legacy controllers. We have basic container which are available in these controllers, they are built thanks to `PrestaShop\PrestaShop\Adapter\ContainerBuilder` which "manually" build a container with a few services defined in `config/services/front|admin`.
We can extend this builder so that it integrate `Doctrine\Bundle\DoctrineBundle\DependencyInjection\DoctrineExtension` thus allowing access to all doctrine services.

### Pending PR

You can find more details about the implementation in this PR: https://github.com/PrestaShop/PrestaShop/pull/12564

## Consequences

This would extend PrestaShop feature and allow developpers used to Doctrine to use this component, and could slowly get other ones to use tools available in Symfony.

### Potentiel drawback

Few potential problems for back-office, the question is for front-office: Does the definition of `DoctrineExtension` have a negative effect on performance (when not used)? Since it's only the definition and services are not instanciated as long as they are not used it should have minimum influence.
