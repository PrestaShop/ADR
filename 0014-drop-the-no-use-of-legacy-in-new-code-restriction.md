# 14. Drop the "No Use Of Legacy In New Code" restriction

Date: 2021-08-25

## Status

In discussion

## Context

The [original problem]((https://build.prestashop.com/news/new-architecture-1-6-1-0/) was that PrestaShop was made out mainly of static classes, and there was no dependency injection. To adress that problem, it was decided that non namespaced code would be progressively refactored into a `Core` namespace, which would only contain code using with dependency injection. Furthermore, Core code wouldn't be allowed to depend directly on non namespaced classes, but it could to it by the means of `Adapter` classes that would act as a bridge between new and old code.

The "no direct dependency between Core and Legacy" rule led to an ever growing collection of adapters that resulted in greatly increased code complexity and duplication. In some cases, the same service can have a legacy, adapter and core implementations, with subtle differences between each one. Furthermore, the constraints of backward compatibility further increase the difficulties to refactor code into Core, because the suface of the "public API" is larger.

## Decision

The following decision applies to both `Core` and `PrestaShopBundle` classes (referred as to "Core" for shortness):

1. Core classes MAY now depend on instances of legacy classes, provided the following rules are respected:

	- Legacy classes MAY be used either as injected parameters or constructed within, but caution must be exerced when using legacy classes that produce side effects, have global state or don't guarantee internal consistency. In those cases, delegating access through dedicated services is still recommended

	- Core classes MUST NOT call static methods on other classes, except for factory methods.

	- Core classes MUST rely on Application-level services, Repositories or Data Providers as necessary in order to encapsulate access to data provided by static classes or methods.

2. Core classes MUST NOT reimplement code found in legacy classes, without deprecating the original method/class and making it rely on the new code.

3. The Adapter namespace will be phased out:

	- Classes in the Adapter namespace will be copied to the Core namespace.

	- The original Adapter classes will be emptied out, made to extend the Core classes, and deprecated so that they can be fully removed in the next major.

	- Adapter services must be deprecated and copied into the core namespace as well.

	- No code must depend on Adapter classes or services.

## Consequences

Refactoring will become easier, and complexity and code duplication will be reduced. On the other hand, it will be harder to refactor legacy classes without introducing breaking changes.
