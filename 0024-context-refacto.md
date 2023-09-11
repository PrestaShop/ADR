# 24. Context refactoring, replacement of the legacy context

Date: 2023-02-21

# Status

In Progress

# Context (not a pun)

In legacy code of PrestaShop, but sadly in a lot of recent code as well, we rely heavily on the `Context` [class/singleton](https://github.com/PrestaShop/PrestaShop/blob/8.1.0/classes/Context.php#L316)
This legacy context, although convenient at times, has many flaws:
- it's completely mutable, and lots of bugs in PrestaShop are related to values in this context being changed by unexpected code
- the way it's built is not very well known mostly because it's fragmented in many different places in the current code
- it lacks validation and exception feedback, so we have no clear way of knowing the Context is badly setup, resulting in hours/days of debugging to understand the root cause sometimes
- we have no equivalent of this principle in Symfony, so we are forced to rely on this legacy code even in modern implementation
- we do have a `LegacyContext` service, but it's merely an accessor to the `Context::getContext` static method

The principle of having contextualized data is not a bad thing, it's even required for many use cases. However, we should build our code to be as independent as it can be from the context and prioritize stateless code that rely on parameter instead of global states, the need of contextualized is still real especially in a browser session.

The purpose of this ADR is to define a new architecture that intends on replacing the legacy context in our modern code, it should aim at removing/reducing the current drawbacks. Ultimately, this new architecture should completely replace the old one or at least be responsible for building the legacy one for backward compatibility.

# Moder context architecture

## Split contexts

The first decision about this new architecture is to split the current Context that contains everything. We will split it into multiple sub contexts, this way you can only inject the relevant part in your services and not all of it.
If sub contexts are split it also means they can be built independently thus optimizing their build process and allowing process where only one would be needed and actually built. We identified these main sub contexts but the list may increase in the future:

- Shop context
- Language context
- Currency context
- Country context
- Employee context
- API Client context (new one)

## Context building

Each sub context will be built upon three main components:
- a `SubContext` class/service with getters/function allowing access to the data, the data must be immutable to ensure it remains unchanged during each process/request, this class is basically an immutable DTO but it can include some methods if needed (ex: `EmployeeContext::hasAuthorization`)
- a `SubContextBuilder`, it provides getters/setters to specify the parameters required to build the `SubContext` class and a `build` method that returns the `SubContext` instance
  - any context builder can also implement the `PrestaShop\PrestaShop\Core\Context\LegacyContextBuilderInterface` interface, it requires implementing `buildLegacyContext` that will be used to initialize the legacy Context for backward compatibility, of course the data used to build the legacy context must be synced with the one used for the modern Context service
  - the builder is in charge of fetching any data from DB or any advance building operations However it is not capable of detecting itself the required arguments to build the context (like Entity IDs), this responsibility is left to the listener
- a Symfony listener per sub context, it is responsible for getting the data required and inject it inside the `SubContextBuilder`, nothing more they are not responsible for triggering the actual build
  - the data initialized by the listener must be kept to its minimum, like a Locale code, or an Entity ID All the advanced fetching is left to the builder, the listener is also responsible for defining the default fallback when they are needed
  - There can be multiple listeners for a same sub context it will allow adapting the initialization depending on the use case or environment (ex: one for Back Office, one for OAuth API, one for CLI commands, one for FrontOffice one day, ...)

Based on these three elements we can then use the Symfony DI in our favor, the `SubContext` is a service, so it will be easily injectable any place we need it, the `SubContextBuilder` is the factory that allows to build this service.

The `SubContext` service **MUST** be a [**lazy** service](https://symfony.com/doc/current/service_container/lazy_services.html) for several reasons:
- it improves performance (even if the cost is minimum) as it won't be built unless it's actually used
- it is built as late as possible, only on the first time it's actually used This way it leaves more time in the process to override/change the values set by the initial listeners thus allowing more flexibility than an instant building
- it will solve some DI issues we have so far when the services depending on context are built on each request even if the data is not available, resulting in errors for code that is not even run

## Context DTO

- The `SubContext` DTO services are immutable objects
- All their fields must be readonly (we can use PHP8 `readonly` now) but should be kept as private
- The fields are accessible via getter methods
- **Required** contexts that always have to be accessible (like `ShopContext`, `CountryContext` are mandatory and always populated even if it requires using a fallback value) are direct DTOs, meaning there is no extra layer they represent the `SubContext` data directly
- **Optional** contexts can be empty and not possible to initialize sometime (like `EmployeeContext` that cannot be created when the user is logged out, or when the core is used via the OAuth API, in this case `ApiClientContext` is the equivalent) should have an extra DTO, the data is stored in a sub DTO and this field is optional/nullable in the `SubContext` service

### Example for Country context (required context)

Here are examples on this architecture divided into three elements for the `Country` context
- [CurrencyContext](https://github.com/PrestaShop/PrestaShop/blob/0cff81b074427b3768ae3cde6fe9a3178d08206a/src/Core/Context/CurrencyContext.php)
- [CurrencyContextBuilder](https://github.com/PrestaShop/PrestaShop/blob/118859f73728aa7e3c4bd59a5bebf8cbb012c62d/src/Core/Context/CurrencyContextBuilder.php)
- [CurrencyContextListener](https://github.com/PrestaShop/PrestaShop/blob/b41e76d7ba24248a004fb58ea370b35b3f8e7071/src/PrestaShopBundle/EventListener/Context/Admin/CurrencyContextListener.php)

You'll notice that:
- the `CurrencyContext` class forces setting all the data and therefore allows accessing all the fields straight away
- the `build` methods triggers an Exception when it's called while not having the required data, since this context is required we must fail early in the workfow to indicate that the initialization failed
- the Listener is run very early in the request (on kernel event), even if the context is mandatory it shouldn't trigger an exception in case it cannot find the data because there could be another listener after it that will

### Example for Employee context (optional context)

Here are example of an optional sub context for `Employee`
- [EmployeeContext](https://github.com/PrestaShop/PrestaShop/blob/79dc47622b48c214c77f7d14db8ff2aa0afa4c2e/src/Core/Context/EmployeeContext.php)
- [Employee sub DTO](https://github.com/PrestaShop/PrestaShop/blob/26fc41f78546eaf53a04f494f9c731c0142cb65f/src/Core/Context/Employee.php)
- [EmployeeContextBuilder](https://github.com/PrestaShop/PrestaShop/blob/33574f7a45bc595831f2e75251b50a182f09c6ca/src/Core/Context/EmployeeContextBuilder.php)
- [EmployeeContextListener](https://github.com/PrestaShop/PrestaShop/blob/3ad7d8943ae9ddc1b62479895f82ba74956d3670/src/PrestaShopBundle/EventListener/Context/Admin/EmployeeContextListener.php)

You'll notice that:
- the context has a nullable DTO field, allowing to check in the code if an Employee context is currently available
- the sub context DTO follows the same rules as mandatory contexts, it's an immutable DTO with all the fields accessible on first level
- the Builder's `build` method can be called even when the employee ID has not been set, it will not trigger an exception and will simply skip the build phase

# Context dependency replacement

Once these new sub contexts are available it's time to use them as such in new code:
- no usage of `LegacyContext` is allowed anymore
- no usage of `Context::getContext` is allowed anymore
- if it turns out the sub context is missing some data that is only accessible via the legacy one then the sub context must be enriched or a new one may be needed
- when only a part of a sub context is needed in a service (like Language iso code) we should still inject the whole sub context, not just the small needed part

We will also have to start a refacto on all the current code that rely on `LegacyContext` and `Context`, especially we'll be able to fix many improper dependency injections like:

```yaml
my_service:
  - '@=service("prestashop.adapter.legacy.context").getContext().shop.id'

# Will be replaced by (of course autowiring is possible and even encouraged), the internal code will also have to be adapted
my_better_service:
  - '@PrestaShop\PrestaShop\Core\Context\ShopContext'
```

A lot of code will have to be adapted but for the better, most of these changes will actually impact the classes' constructor adn internal code but not their public behavior and/or methods which is in line with our backward compatibility policy (classes that are only/mostly used as service can have their constructor modified), but of course most of these changes should be done in major version, ideally, as they can be breaking changes.

# Legacy context building

These new sub contexts will bring more stability in the context building, their architecture will also naturally centralize the building process and offer more control on this highly unstable key component. So we could also benefit from it in places still relying on legacy Context.
To do this the `SubContextBuilder` **can** implement an alternate interface `LegacyContextBuilderInterface` with a single `buildLegacyContext` method, since the builder is already able to build the new sub context based on the parameters that it was provided with it can also correctly set/fill the legacy Context.
A dedicated listener will be in charge of looping through all the builders that implement `LegacyContextBuilderInterface` and build the legacy context automatically, as long as the required data has been provided.

Thanks to this new legacy Context building we'll be able to remove legacy code piece by piece in the modern code and Symfony pages since we don't rely on it anymore, the legacy context would still be correctly built though because some of the core code still relies on it until it's refactored to rely on the new sub contexts, but also mainly to keep backward compatibility especially for modules.

Once this backward compatibility layer is implemented for a sub context we can consider removing/cleaning all the legacy code that built the legacy context in the first place. To do this we may have to split the internal config from Front office and back office because many things, for example, are done in included files like [config.inc.php](https://github.com/PrestaShop/PrestaShop/blob/a253d05fa6882b754ebb4c49e2528c28d2087056/config/config.inc.php)
and are used in both FO and BO. The new context are handled by Symfony, so they're not available in FO yet.
