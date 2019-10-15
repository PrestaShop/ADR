# 3. Use of autowiring

Date: 2019-10-15

## Status

Rejected

## Context

Symfony provides a very useful tool called [Autowiring](https://symfony.com/doc/3.4/service_container/autowiring.html). It allows to magically bind classes and their dependencies as long as both are declared as services, and the dependencies are declared using their FQCN as service identifier.

Advantages:

- Less boilerplate configuration code for every service as you don't have to manually bind dependencies manually.

Disadvantages:

- Dependencies must be declared using the FQCN instead of a service identifier like "prestashop.core.foo.bar".
- Currently existing services would have to be aliased in order to have service names follow the required naming convention for autowiring. This would lead to confusion as to which service name use in code, and in case a module wanted to replace them, they would have to replace both.
- Dependencies type-hinted as interfaces can have one and **only one** implementation for autowiring to work.

## Decision

Activiting autoriwing is rejected for the 1.7 version.

## Consequences

Services will have to continue being wired manually.
