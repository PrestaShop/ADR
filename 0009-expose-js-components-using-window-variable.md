# 9. Expose js components using window variable

Date: 2020-08-20

## Status

Accepted

## Context

In order for modules to use JavaScript components from the Core, they need to import them using statements like:

```
// in order to use translatable type
import TranslatableInput from '../../../../../admin-dev/themes/new-theme/js/components/translatable-input';
```

This path is not robust, makes CI/CD harder, and also is not compatible with some development environments using symlinks or containers.

## Decision

We have decided about a system which resolves in 4 concepts:

1. Reusable components in BO will be available globally through `window.prestashop` (name can still be modified in short term).

All PrestaShop components will be bundled together and made available in all pages using this mean. Each controller decides which components it chooses to initialize.

2. Reusable components will be available as a namespace `window.prestashop.component`.

The namespace will contain classes like this `prestashop.component.SomeComponent`. If you want to get a new instance of `SomeComponent`, you call `new prestashop.component.SomeComponent(...params)`

3. Reusable components will be available as initialized instances through `window.prestashop.instance`. These instances are initialized with default parameters by the `initComponents` function.

4. A function `initComponents` available through `prestashop.component` is responsible for building `window.prestashop.instance`.

### Why a namespace and a collection of instances

Since you have access to both constructors and components, developers are free to choose how to initialize and control their components.

If you don't want to initialize a given component with default parameters, you can always call `new prestashop.component.SomeComponent(...myOwnParameters)`.

If you need to apply some mutation to an already initialized component, you just get the global instance: `prestashop.instance.someComponent.doSomething(...)`.


## First implementation

To be completed once we have validated the related PR https://github.com/PrestaShop/PrestaShop/pull/20591

## Consequences

What becomes easier :

- As a module developer, you'll be able to use every global javascript features of the core (if they are exposed inside the window object)
- As a module developer, you're no longer forced to use hard paths breaking your dev environment or CI/CD workflow
- As a core developer, you're no longer forced to import every modules you need, if they are exposed in the window object, you'll be able to use them directly
- We now provide a way to avoid multiple executions of the same feature, while before, every modules was using them as they wants without caring of other modules. It means that we'll maybe avoid some side effects on some features.
- Modules will be able to override core components without breaking the shop update workflow

What could become harder :

- It will require more knowledge about the project. Before, you needed to know that TranslatableInput exist, while you'll need to also now that they are global and don't require to be imported at all.
