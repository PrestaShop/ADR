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
ApiPlatform resource class: `PrestaShopBundle\ApiPlatform\Resources\Hook`

URI conventions, we use the domain word as a base for the URI, to follow REST conventions we use the plural (even for single entity endpoints).
For the identifier related to one entity we use the domain name suffixed by `Id` (ex: Hook -> `hookId`).

List: `PrestaShopBundle\ApiPlatform\Metadata\PaginatedList` `/hooks`<br />
One hook endpoint: `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet` `/hooks/{hookId}`

For APIs that are sub part of a larger entity (whether it's to display its related entities or small parts of a big entity) we keep the initial domain name as the beginning
of the URI and append it with the definition of the sub parts (separated by a `/`). If it's the sub part of an identified entity, we complete the initial URI path and append
the sub part after the entity ID. When the sub part is an action name or a compound name we use kebab case convention.

Example:
- Hook status (sub part): `/hooks/{hookId}/status`
- Search (action) products:  `/products/search`
- Product combinations (sub part): `/products/{productId}/combinations`
- Assign product to category (action): `/products/{productId}/assign-to-category`
- Set product carriers (set a sub part): `/products/{productId}/carriers`
- Attribute group (list):  `/attribute-groups`
- Attribute group associated values (list of sub part):  `/attribute-groups/{attributeGroupId}/attributes`

### Multilang

Some entities in PrestaShop have multilang values (like product names, category descriptions, ...), this data must be presented in the API endpoints:
- single entities endpoints return ALL the languages in an associative array indexed by locale (ex: `{"names": {"en-US": "english name", "fr-FR": "nom français"}}`)
- list of entities return only one language so multilang fields are returned as strings (ex: `{"name": "english name"}`), the language used by default is the default language configured on the shop, but you can specify a `langId` query parameter to fetch another language values

To allow knowing the association between languages and language IDs a `/languages` endpoint will be accessible without any needed permission.

### HTTP methods

Read operations use **GET** method and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet`<br />
Creation operations use **POST** method (without ID specified) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate`<br />
Full update operations use **PUT** method (ex: PUT `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSUpdate`<br />
Partial update operations use **PATCH** method (ex: PATCH `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSPartialUpdate`<br />
Delete operations use **DELETE** method (ex: DELETE `/products/{productId}`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSDelete`<br />
Duplicate operations use **POST** method with ID specified (ex: POST `/products/{productId}/duplicate`) and should use `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate`

### Custom operations

To simplify the integration of our custom CQRS based implementation some custom operations were developed, they must be used in the core endpoints to remain consistent.

- `PrestaShopBundle\ApiPlatform\Metadata\CQRSGet` for read operations on a single resource
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSGetCollection` for read operations on a list of resources (not paginated)
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate` for creation and duplication operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSUpdate` for full update operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSPartialUpdate` for partial update operations
- `PrestaShopBundle\ApiPlatform\Metadata\CQRSDelete` for delete operations
- `PrestaShopBundle\ApiPlatform\Metadata\PaginatedList` to paginate elements

### Bulk operations

Similar convention for list of IDs we use the domain and append `Ids` at the end, bulk endpoints always use `bulk-` as a prefix for the action they are doing (even if the HTTP method implies it).

- Bulk delete products: Method DELETE `/products/bulk-delete` with `productIds` parameter in request body (array of product IDs)
- Bulk duplicate products: Method POST `/products/bulk-duplicate` with `productIds` parameter in request body (array of product IDs)
- Bulk update status: Method PUT `/products/bulk-update-status` with `productIds` parameter in request body (array of product IDs)

In case no HTTP method finds a consensus for the bulk operation, POST method is used by default.

### Scope names

This convention is only for the Core API, if modules wish to define their own API they are free to use any convention for the scopes.
We use the entity domain name in its singular form, and append the action, the whole string is written in snake case.
Each scope is supposed to represent a single authorized action.

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
