# 23. CQRS API guidelines

Date: 2023-08-24

## Status
Accepted

## Context

In PrestaShop 9.0, [a new API](https://github.com/PrestaShop/PrestaShop/pull/29931) is being built powered by ApiPlatform, based on Commands (from CQRS). We need
to define a convention for the path format and the ApiPlatform resources classes. We will mostly follow REST conventions but it's useful to write down clearly a few
things to avoid confusion later.

## Convention

### URI path and parameters conventions

We base the naming on the domain from CQRS (which usually matches the ObjectModel entity name as well) in `PrestaShop/PrestaShop/Core/Domain`

Example: Hook

Domain: `PrestaShop/PrestaShop/Core/Domain/Hook`<br />
ApiPlatform resource class: `PrestaShop\Module\APIResources\ApiPlatform\Resources\Hook` in the `ps_apiresources` module, where the resources live (one folder per domain, ex: `Resources\Customer\Customer`); the core only keeps the resources that must work without the module (`Language`, `ApiClient`). The module's `.ai/CONTEXT.md` holds the implementation rules, this ADR holds the contract.

URI conventions, we use the domain word as a base for the URI, to follow REST conventions we use the plural (even for single entity endpoints).
For the identifier related to one entity we use the domain name suffixed by `Id` (ex: Hook -> `hookId`).

List: `PrestaShopBundle\ApiPlatform\Metadata\PaginatedList` `/hooks`<br />
One hook endpoint: `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet` `/hooks/{hookId}`<br />
List from a CQRS query that paginates: `PrestaShopBundle\ApiPlatform\Metadata\CQRSPaginate` `/products/{productId}/combinations`<br />
List without pagination: `PrestaShopBundle\ApiPlatform\Metadata\CQRSGetCollection` `/customers/{customerId}/carts`

For APIs that are sub part of a larger entity (whether it's to display its related entities or small parts of a big entity) we keep the initial domain name as the beginning
of the URI and append it with the definition of the sub parts (separated by a `/`). If it's the sub part of an identified entity, we complete the initial URI path and append
the sub part after the entity ID. When the sub part is an action name or a compound name we use kebab case convention.

Example:
- Hook status (sub part): `/hooks/{hookId}/status`
- Search (action) products:  `/products/search`
- Product combinations (sub part): `/products/{productId}/combinations`
- Assign product to category (action): `/products/{productId}/assign-to-category`
- Set product carriers (set a sub part): `/products/{productId}/carriers`
- Sub-entities of a domain nest under the domain root: attribute groups (list) `/attributes/groups`, one group `/attributes/groups/{attributeGroupId}`, its values (list of sub part) `/attributes/groups/{attributeGroupId}/attributes`, one attribute `/attributes/attributes/{attributeId}`

### Multilang

Some entities in PrestaShop have multilang values (like product names, category descriptions, ...), this data must be presented in the API endpoints:
- single entities endpoints return ALL the languages in an associative array indexed by locale (ex: `{"names": {"en-US": "english name", "fr-FR": "nom français"}}`), at every level: the localized fields of nested objects are indexed by locale too
- list of entities based on a grid return only one language, since a grid lists one language, so multilang fields are returned as strings (ex: `{"name": "english name"}`), the language used by default is the default language configured on the shop, but you can specify a `langId` query parameter to fetch another language values
- list of entities based on a CQRS query follow the contract of the query: when the query returns all the languages, the endpoint returns them too, like a single entity endpoint

To allow knowing the association between languages and language IDs a `/languages` endpoint will be accessible without any needed permission.

The other contexts are selected the same way, by request parameters: `shopId`, `shopGroupId`, `shopIds` or `allShops` set the shop constraint, `currencyId` the currency (defaults: the default shop, language and currency of the shop). A list returns the values of the selected shop and currency, and of the selected language when it returns only one.

### HTTP methods

Read operations use **GET** method and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet`<br />
Creation operations use **POST** method (without ID specified) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate`<br />
Full update operations use **PUT** method (ex: PUT `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSUpdate`<br />
Partial update operations use **PATCH** method (ex: PATCH `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSPartialUpdate`<br />
Delete operations use **DELETE** method (ex: DELETE `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSDelete`<br />
Duplicate operations use **POST** method with ID specified (ex: POST `/products/{productId}/duplicate`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate`

The operation follows the command: **PATCH** only when the command accepts partial data (optional setters, nullable parameters); when every parameter is required on each call the operation is a **PUT**, even when it only covers a part of the entity (ex: PUT `/carts/{cartId}/addresses`).<br />
A delete and a body-less action answer **204**; every other write returns the updated resource, read back through the query of the resource.<br />
An operation receiving files uses **POST** with a `multipart/form-data` body, even when it updates: PHP only reads the files of a POST request. An update that may carry a file is therefore a multipart **POST**, partial or full like its command; when it is partial, a JSON **PATCH** on the same URI updates the fields without the file, which is left out of its input (a full update needs the file, so it has no JSON variant). A creation is a **POST** anyway: it accepts JSON, and multipart when a file is sent. In a multipart body the other fields are form fields: booleans are sent as `1` / `0`, nested values with the bracket syntax (ex: `names[en-US]=...`) or, from PrestaShop 9.2.0, as a JSON-encoded part.

### Custom operations

To simplify the integration of our custom CQRS based implementation some custom operations were developed, they must be used in the core endpoints to remain consistent.

- `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet` for read operations on a single resource
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSGetCollection` for read operations on a list of resources (not paginated)
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate` for creation and duplication operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSUpdate` for full update operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSPartialUpdate` for partial update operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSDelete` for delete operations
- `PrestaShopBundle\ApiPlatform\Metadata\PaginatedList` to paginate elements through a back-office grid data factory
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSPaginate` (PrestaShop 9.0.3) to paginate elements through a CQRS query that handles the pagination itself (`itemsField` / `countField` of the query result)
- `PrestaShopBundle\ApiPlatform\Metadata\UpdatePosition` to reorder elements

Both paginated operations share one contract: the query parameters `limit`, `offset`, `orderBy`, `sortOrder` and `filters[field]` (API field names, translated to the grid or query fields by `filtersMapping`, defaults from `filtersClass`) and the response envelope `{totalItems, orderBy, sortOrder, limit, offset, filters, items}`, `items` always being a JSON list. `CQRSGetCollection` returns a plain JSON list that is neither filtered nor paginated: a list query that paginates uses `CQRSPaginate`.

### Bulk operations

Similar convention for list of IDs we use the domain and append `Ids` at the end, bulk endpoints always use `bulk-` as a prefix for the action they are doing (even if the HTTP method implies it).

- Bulk delete products: Method DELETE `/products/bulk-delete` with `productIds` parameter in request body (array of product IDs)
- Bulk duplicate products: Method POST `/products/bulk-duplicate` with `productIds` parameter in request body (array of product IDs)
- Bulk update status: Method PUT `/products/bulk-update-status` with `productIds` parameter in request body (array of product IDs)

A bulk update is a **PUT**: its command needs the IDs and the value on every call (see HTTP methods). In case no HTTP method finds a consensus for the bulk operation, POST method is used by default.

The list of IDs is required (an empty body is a 422) and each ID is a positive integer (422 otherwise). When the handler collects the errors (PrestaShop 9.1.2, `AbstractBulkCommandHandler`), a partial failure answers **207 Multi-Status** with one error per failed ID. A bulk endpoint may take an option in the path (ex: DELETE `/categories/bulk-delete/{deleteMode}`).

### Status codes

- **200** for a read, an update or an action with a response, **201** for a creation, **204** for a delete or a body-less action
- **404** for an unknown identifier, in the path or in the body (`shopIds`, `customerId`...)
- **422** for a validation error of the resource constraints or a domain constraint error (`GroupConstraintException`...), as RFC 7807 problem details, the constraint violations listed per field
- **207** for a partial failure of a bulk operation
- **403** for a token without the scope of the operation

An unmapped exception is a 500: every domain exception an endpoint can raise is mapped to its status (`exceptionToStatus`).

### Scope names

This convention is only for the Core API, if modules wish to define their own API they are free to use any convention for the scopes.
We use the entity domain name in its singular form, and append the action, the whole string is written in snake case.
Each scope is supposed to represent a single authorized action. Every operation declares at least one scope; the only unscoped endpoints are `/languages` and `/api-clients/infos`, needed by every client.

Basic Example:
- Orders read operations: `order_read`
- Orders write operations: `order_write`

In the future we may define some more detailed sub scope, but they should follow a similar convention:

Example:
- Orders update address: `order_update_address`
- Orders create invoice: `order_create_invoice`

### API resources properties

- all the class fields must be strictly typed
- the localized properties **do not** start by `localized` (ex: `public array $publicNames;` not `public array $localizedPublicNames;`) and should use the `PrestaShopBundle\ApiPlatform\Metadata\LocalizedValue` PHP attribute to have the automatic ID->locale conversion
- the boolean fields don't need to start by `is` which is redundant with the type (ex: `public bool $ready;` not `public bool $isReady;`)
- the status of the entities is not always expressed the same way (active, enable, enabled, ...) we want to homogenize this field by always using `public bool $enabled;`
- you should use the internal `CQRSQueryMapping`, `CQRSCommandMapping`, `ApiResourceMapping` instead of the `SerializedName` attribute (because it is not applied everywhere appropriately for the documentation)
- the identifier property is named like the URI parameter (`hookId`) and declared with `#[ApiProperty(identifier: true)]`
- a nested object is a typed DTO class (ex: `public CustomerOrdersInformation $ordersInformation;`, a list of objects with `@var CustomerOrderRow[]`, a list of scalars with `@var int[]`), never a free-form `array`
- a property is nullable only when the core returns null for it, and the read format of a property is also its write format
- an endpoint relying on a feature of a newer core declares `minVersion`: it is absent (404, not documented) on older cores instead of failing
