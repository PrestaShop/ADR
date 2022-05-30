# 18. Horizontal migration

Date: 2022-03-24

## Status

Approved

## Context

In our current migration strategy, "migrating a page to Symfony" is not just migration: it's actually a migration + refactoring. This includes switching the view from Smarty to Twig, switching listings from HelperList to Grid, switching forms from HelperForm to Symfony forms, applying the CQRS layer, and more. 

This process takes a very long time, and as a result, the Back office structure has become heterogeneous, with some pages fully Symfony-based and others full legacy-based, increasing developer and performance overhead. 

In addition, legacy subsystems must remain in place and continue being maintained, they cannot be removed as long as there are legacy pages in PrestaShop. And we still have years' worth of work ahead of us to finish. Let's imagine that migrating the rest of the Back office to Symfony using the current strategy will take, say, 4 years. This means that 2 years from now, we will still have about 25% of the pages running on the legacy architecture. And because of it, we will still have to maintain AdminController, Dispatcher, overrides, etc for the whole time, until the migration is finished – 4 years from now.

## Decision

To accelerate the phasing-out of legacy subsystems, the strategy to migrate the remaining pages should be changed. Instead of migrating and refactoring whole pages in a single shot, we should divide the migration into stages, each covering a different technical layer, and deliver a single stage of migration for each page at a time. 

We have identified 4 migration stages:

1. Controller layer
2. View layer
3. Form layer
4. CQRS layer

Each time we migrate a page, we only migrate the part corresponding to the first stage. Only once the first stage has been performed on all the remaining pages, we can move to the second stage and perform it on all the pages left to migrate. This repeats until all the stages are complete, and the migration is finished.

We call this strategy "horizontal migration", in opposition to our previous strategy, referred to "vertical migration", because it focuses on finishing whole technical layers across all pages before moving on to the next layer. The end result is the same: full migration to Symfony. It's the path to get there that changes.

### First stage: migrate the "Controller layer"

The first stage consists in migrating the Controller Layer to Symfony (based on FrameworkBundleAdminControllers, services, sf router, and action-methods), but keeping most of the original behavior as it is, within the controller itself. This includes maintaining the Smarty template engine as well as closely related legacy components like HelperList, HelperForm, and others. 

A new namespace `Bridge` will be created within the PrestaShopBundle to contain intermediary code that will be removed as the migration progresses. For example, temporary traits will be added into horizontally migrated Syfony controllers, enabling any necessary legacy features which do not belong in a fully migrated Symfony controller (like Smarty support). These traits are meant to be removed when the features will no longer be required.

The `LegacyControllerBridgeInterface` replaces the controller that is present in Context, providing methods needed by modules but that don't belong in Symfony controllers, including:

- `setMedia`
- `addCss`
- `addJs`
- `addJqueryPlugin`
- `addJqueryUI`

It will also provide access to public fields that were accessible from the `AdminController` class to keep as much backward compatibility as possible:

- `php_self`
- `controller_name`
- `table`
- `meta_title`
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

### Why this decision makes sense

Suppose there are 30 pages left to migrate. Instead of creating 30 PRs, each containing 10000 lines changed in 80 files, taking forever to complete, review and verify, we will create four groups of 30 PRs. The first 30 PRs only modify the page's controller, leaving the rest of the code untouched. The second 30 PRs modify only the page's templates (switching them to Twig). The third 30 PRs introduce Symfony forms and Grid. And the fourth 30 PRs add CQRS. Each "group" of 30 PRs cannot be started before the previous groups has been finished.

Compared to the vertical approach, the horizontal approach might take slightly longer to complete. However, its main advantage is that we no longer have to wait for the whole migration project to be complete before being able to remove obsolete subsystems: as soon as a stage is complete, developers benefit from a single way of doing things across all the system, and all the obsolete legacy subsystems can be removed. For example, once all controllers have been migrated to Symfony, Dispatcher and AdminController are no longer needed and can be removed (or deprecated), and developers benefit from the Symfony debug bar everywhere. By reducing the scope of migration deliverables, these "events" will happen a lot sooner than if we had to wait for the whole project to be finished: following the previous hypothetical duration of 4 years to finish the whole migration, we could get rid of AdminController and Dispatcher in two years, instead of the four it would take using the vertical approach. Same with Smarty: we could have it removed in three years instead of four. 

Reducing the perimeter of deliverables in smaller chunks means smaller PRs, which should translate in easier and faster reviews and QA validation.

We can still continue using the vertical approch for pages where the horizontal approach would not make tactical sense, either because they are too simple to benefit from the advantages of horizontal migration, or because they require extensive refactoring (eg the product page).
