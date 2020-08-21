# 9. Expose js components using window variable

Date: 2020-08-20

## Status

In discussion

## Context

Modules are unable to use translatable type without using hardlinks such as:

```
import TranslatableInput from '../../../../../admin-dev/themes/new-theme/js/components/translatable-input';
```

This path is making CI/CD harder, and also break on some development environment such as symlinks or containers.

## Decision

Reusable components in BO will be available globally in `window.prestashop.components`, like this: `window.prestashop.components.TranslatableInput.init()`.

This means that if they are already initiated, they need to avoid a new execution.

Currently, the `prestashop` object is not used in the BO, but as we want to stick to the workflow used on FO, we would like to introduce it in the backoffice.

@JevgenijVisockij submited a way of initializing global objects :

```
const initComponents = () => {
  let translatableInput = null;
  window.prestashop = {
    components: {
      translatableInput: {
        init() {
          if (translatableInput === null) {
            translatableInput = new TranslatableInput();
          }
          return translatableInput;
        },
      },
    }
  }
}
```

and used like :

```
$(() => {
    window.prestashop.components.translatableInput.init();
});
```

and maybe you could get the possibility to access to the component directly like :

```
$(() => {
    new window.prestashop.components.translatableInput.component({localeItemSelector: '.my-selector'});
});
```

based on a little POC: [20313](https://github.com/PrestaShop/PrestaShop/pull/20313)

This means that we'll have to refacto a lot of javascripts in order to port every components into globals, or at least begin by mosts used ones.

## Consequences

What becomes easier :

- As a module developer, you'll be able to use every global javascript features of the core (if they are exposed inside the window object)
- As a module developer, you're no longer forced to use hard paths breaking your dev environment or CI/CD workflow
- As a core developer, you're no longer forced to import every modules you need, if they are exposed in the window object, you'll be able to use them directly
- We now provide a way to avoid multiple executions of the same feature, while before, every modules was using them as they wants without caring of other modules. It means that we'll maybe avoid some side effects on some features.
- Modules will be able to override core components without breaking the shop update workflow

What could become harder :

- It will require more knowledge about the project. Before, you needed to know that TranslatableInput exist, while you'll need to also now that they are global and don't require to be imported at all.
