# 14. Drop the "No Use Of Legacy In New Code" restriction

Date: 2021-08-25

## Status

Accepted

## Context

Originally, PrestaShop was made out mainly of static classes, with no dependency injection. To address that problem, it [was decided](https://build.prestashop.com/news/new-architecture-1-6-1-0/) that non namespaced code would be progressively refactored into a `Core` namespace, which would only contain code using with dependency injection. Furthermore, Core code wouldn't be allowed to depend directly on non namespaced classes, but it could to it indirectly by the means of `Adapter` classes that would act as a bridge between new and old code.

The "no direct dependency between Core and Legacy" rule led to an ever-growing collection of adapters, which resulted in greatly increased code complexity and duplication. In some cases, the same service can have a legacy, adapter and core implementations, with subtle differences between each one. Furthermore, the constraints of backward compatibility further increase the difficulties to refactor code into Core, because the surface of the "public API" is larger.

## Decision

The following decision applies to both `Core` and `PrestaShopBundle` classes (referred as to "Core" for shortness):

1. **All new Core classes SHOULD be placed either in the `Core` or the `PrestaShopBundle` namespace**, following on the rules established previously.

   - New classes MUST NOT be added to the `Adapter` namespace, and SHOULD NOT be added to the legacy (root) namespace.

2. **Core classes MAY depend on instances of legacy classes**, provided the following rules are respected:

    - Legacy classes MAY be used either as injected parameters or constructed within, but caution must be exerted when using legacy classes that produce side effects, have global state or don't guarantee internal consistency. In those cases, these classes SHOULD be accessed through dedicated services which enforce consistency.

    - Core classes MUST NOT call static methods on other classes, except for factory methods, stateless tool methods, or within services dedicated to encapsulate a static class.

    - Core classes MAY access to data provided by static classes or methods static classes by relying on dedicated services (Application services, Repositories, Data Providers...).

3. **Core classes MUST NOT reimplement code found in legacy classes**, without deprecating the original method/class (and optionally, making it rely on the new implementation).

4. **The Adapter namespace MUST be phased out** eventually:

    - Classes in the Adapter namespace MUST be copied to the Core namespace.

    - The original Adapter classes MUST be emptied out, made to extend the Core classes, and deprecated so that they can be fully removed in a following major.

    - Adapter services MUST be deprecated and copied into the core namespace as well.

    - Code MUST NOT depend on Adapter classes or services.

## Consequences

Refactoring will become easier, and complexity and code duplication will be reduced. On the other hand, it will be harder to refactor legacy classes without introducing breaking changes.
