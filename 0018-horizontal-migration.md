# 18. Horizontal migration

Date: 2022-03-24

## Status

Pending

## Context

Instead of choosing one back-office page and applying the whole migration process, switching the view from Smarty to twig, switching listings from HelperList to Grid, switching forms from HelperForm to Symfony forms… we could instead apply one of these changes globally, at once, in all back-office pages.

## Decision

###AdminController

We decide to create a new way to migrate pages to do this. We choose to first migrate the controller to Symfony controller by using some legacy stuff like HelperList, HelperForm, and other helpers.

And we keep the Smarty layer, but we try to adapt the Smarty part to return a Symfony `Response`.

We create a namespace in the PrestaShopBundle named `Bridge`.### AdminController

We will find all the classes in charge of `Controller` in the `AdminController` folder.

We created a `LegacyControllerBridgeInterface` to guarantee that our new controller exposed important methods needed by modules like:

- `setMedia`
- `addCss`
- `addJs`
- `addJqueryPlugin`
- `addJqueryUI`
- ...

There is a folder named `Action`. In this folder, we put all classes to create user action with legacy configuration. There is 4 type of actions:

- `HeaderToolbarAction`
- `ListBulkAction`
- `ListHeaderToolbarAction`
- `ListRowAction`

There is a folder named `Field`. In this folder, we put `Field` class to create field to your list with legacy configuration. A `Field` has a label and an array of configurations.

The `InitControllerListener` initializes the controller by instantiating controller configuration and configuring legacy context, controller, and controller configuration in the `Listener` folder.

The `ControllerConfiguration` class is an object that will contain all needed variables by Smarty to render the header, footer, and others.

The `FilterPrefix` centralizes the way to get the prefix of the filter.

### Helper

We will find legacy helpers in the `Helper` folder but adapt to work with Symfony DI. In the `Helper` folder, you will find:

- `HelperListConfiguration`: this class contains all the configuration needed by the `HelperList` to generate the SQL query and generate the HTML to pass to Smarty to be shown to the client
- `HelperListConfigurator`: this class sets all variables to the `HelperList` from the `HelperListConfiguration` as an `Adapter`
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
