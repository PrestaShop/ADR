# 18. Horizontal migration

Date: 2022-03-24

## Status

Approved

## Context

In our current migration strategy, migrating a page to Symfony requires switching the view from Smarty to Twig, switching listings from HelperList to Grid, switching forms from HelperForm to Symfony forms, applying the CQRS layer, and more. 

As a result of the migration, the Back office structure is heterogeneous, with some pages fully Symfony-based and others full legacy-based, increasing developer and performance overhead. In addition, legacy subsystems must remain in place and continue being maintained, as long as there are legacy pages in PrestaShop. However, we still have year's worth of work ahead of us.

## Decision

To accelerate the phasing-out of legacy subsystems, the migration strategy should be changed: instead of migrating whole pages one by one, we should change the scope as to migrate whole system layers of whole Back office, across all pages. Once a layer has been finished, all obsolete subsystems linked to that layer can be removed, and migration can move on to the next layer.

This project is divided into the following stages, each covering a whole layer:

1. Controller layer
2. View layer
3. Form layer
4. CQRS layer

We call this strategy "horizontal migration", as oppposed to our previous strategy, referred to "vertical migration". The end result is the same: full migration to Symfony. It's the path to get there that changes.

### First stage: migrate the "Controller layer"

The first stage consists in migrating the Controller Layer to Symfony (based on FrameworkBundleAdminControllers, services, sf router, and action-methods), but keeping most of the original behavior as it is, within the controller itself. This includes maintaining the Smarty template engine as well as closely related legacy components like HelperList, HelperForm, and others. 

A new namespace `Bridge` will be created within the PrestaShopBundle to contain intermediary code that will be removed as the migration progresses. For example, temporary traits will be added into horizontally migrated Syfony controllers, enabling any necessary legacy features which do not belong in a fully migrated Symfony controller (like Smarty support). These traits are meant to be removed when the features will no longer be required.

The `LegacyControllerBridgeInterface` replaces the controller that is present in Context, providing methods needed by modules but that don't belong in Symfony controllers, including:

- `setMedia`
- `addCss`
- `addJs`
- `addJqueryPlugin`
- `addJqueryUI`
- ...

#### Helpers

Legacy helpers will have bridges to adapt them to work with Symfony DI. Here are some examples:

- `HelperListConfigurator`: configures a `HelperList` using a `HelperListConfiguration`
- `ResetFiltersHelper` and `FiltersHelper`: these classes come from legacy to handle filters in the list, and the reset of these filters
- `HelperListBridge`: this class is a bridge to use a helper list to render the list in the Controller

#### Smarty, context and layout

Since we are using Smarty and not Twig, we need to handle context initialization and layout features (menu, header, etc) which in vertically migrated controllers are handled through LegacyLayoutAdminController. This will be done through a Kernel event listener called `InitControllerListener`. This means that horizontally migrated controllers will no longer initialize a fake AdminController.

Getting rid of AdminController requires set up a high number of variables that were configured by it and required by the layout. These will be stored in a `ControllerConfiguration` object that will be initialized by configurator classes, each in charge of a setting up different parts of the page:

- `BreadcrumbsAndTitleConfigurator`
- `FooterConfigurator`
- `HeaderConfigurator`
- `ModalConfigurator`
- `NotificationConfigurator`
- `ToolbarFlagsConfigurator`

Finally, controller actions will return a generated HTML Response instance, only using Smarty and the "default" theme instead of Twig and the "new" theme.

### Following stages

This ADR focuses on detailing the first migration stage. The outline of the next stages is as follows.

In the second stage, Smarty will be removed from controllers in favor of Twig. This will require adapting the existing Helpers to work with twig and the new theme, as well as the complete removal of LegacyLayoutController.

The third stage will progressively phase out HelperLists and HelperForms in favor of Grid and Symfony forms, using the form theme and extension features found in vertically migrated controllers.

The fourth and final stage will consist in the introduction of the CQRS layer, removing all logic from controllers. This will be followed by the removal of obsolete components like AdminControllers.
