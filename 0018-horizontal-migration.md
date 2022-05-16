# 18. Horizontal migration

Date: 2022-03-24

## Status

Pending

## Context

In our current migration strategy, migrating a page to Symfony requires switching the view from Smarty to Twig, switching listings from HelperList to Grid, switching forms from HelperForm to Symfony forms, applying the CQRS layer, and more. 

As a result of the migration, the Back office structure is heterogeneous, with some pages fully Symfony-based and others full legacy-based, increasing developer and performance overheaad. In addition, legacy subsistems must remain in place and continue being maintained, as long as there are legacy pages in PrestaShop. However, we still have year's worth of work ahead of us.

## Decision

To accelerate the phasing-out of legacy subsystems, the migration strategy should be changed: instead of migrating whole pages one by one, we should change the scope as to migrate whole system layers of whole Back office, across all pages. Once a layer has been finished, all obsolete subsystems linked to that layer can be removed, and migration can move on to the next layer.

The project is divided into the following stages, each covering a whole layer:

1. Controller layer
2. View layer
3. Form layer
4. CQRS layer

We call this strategy "horizontal migration", as oppposed to our previous strategy, referred to "vertical migration".

### AdminController

The first stage consists in migrating AdminControllers to Symfony while keeping most of the original behavior. This includes the use of legacy components like HelperList, HelperForm, and other helpers.

This also means maintaining the Smarty template engine as it is used in legacy controllers and closely related to legacy helpers.

A new namespace `Bridge` will be created within the PrestaShopBundle to contain intermediary code that will be removed as the migration progresses.

The `LegacyControllerBridgeInterface` replaces the controller that is present in Context, providing methods needed by modules but that don't belong in Symfony controllers, including:

- `setMedia`
- `addCss`
- `addJs`
- `addJqueryPlugin`
- `addJqueryUI`
- ...

### Helpers

Legacy helpers have bridges which adapt them to work with Symfony DI:

- `HelperListConfiguration`: contains all the configuration needed by the `HelperList` to generate the SQL query and generate the HTML to pass to Smarty to be shown to the client
- `HelperListConfigurator`: sets all variables to the `HelperList` from the `HelperListConfiguration` as an `Adapter`
- `ResetFiltersHelper` and `FiltersHelper`: these classes come from legacy to handle filters in the list, and the reset of these filters
- `HelperListBridge`: this class is a bridge to use a helper list to render the list in the Controller

In the `Helper` namespace, we will also create a folder named `HelperListCustomizer` to customize the SQL query for the list by extending the `HelperListBridge` and overriding the `getList` method.

### Smarty

We will find all parts responsible for generating HTML from Smarty in the Smarty folder and return a generated Response instance to the Symfony controller. To generate HTML from Smarty, we need many variables.

To generate all these variables, we create six configurator classes, which are in charge of a part of the page:

- `BreadcrumbsAndTitleConfigurator`
- `FooterConfigurator`
- `HeaderConfigurator`
- `ModalConfigurator`
- `NotificationConfigurator`
- `ToolbarFlagsConfigurator`

The `ControllerConfiguration` object contains the configuration.

We decide to create some traits:

- `SmartyTrait`: contains all methods to handle the HTML rendering like: `renderSmarty`, `setMedia`, `addCss`, `addJs`...
- `AdminControllerTrait`: contains all methods to configure the page and to access services needed by the controller like `addAction`, `addListAction`, `addListField`, `getResetFiltersHelper`, `getFiltersHelper`, `getHelperListBridge`

We must delete this folder when fully migrating all pages to Symfony with CQRS. In this namespace, we will find a Controller folder that contains all needed Controller configurations for Smarty, header, and footer...
