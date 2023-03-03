# 21. Model and Repositories aiming to replace the usages of legacy ObjectModel

Date: 2023-02-23

## Status

In Discussion

## Context

Define interfaces representing the structure of our model without advanced domain intelligence. 

### Decisions to make

1. which folder/namespace to use 

Base folder src/PrestaShop (naming tbd)
- simpler configuration tools and CI
- no PrestaShop\PrestaShop in namespace
- harder for contributors to understand this new structure
- have a new folder with very clean rules, and migrate the Core classes piece by piece into it

Base folder src/Core/Model (naming tbd)
- follow the structure that has been used so far
- tool configuration a bit more complex but it should be possible with exclude/include rules
- time spent to maintain the tools

2. Interfaces object/model

Take all classes based on ObjectModel and turn them into an "entity interface" representing each properties and database columns

### Generic interface (always translated)
Object with multilang properties have getters for those field which return only string already localized (the language id is specified when fetching the object).

```php
interface ContactInterface {
  public function getContactId(): int;
  public function getEmail(): int;

  // Localized getters
  public function getName(): string;
  public function getDescription(): string;
}
```

### Localized interface (mostly for CRUD)

```php
interface LocalizedContactInterface {
    public function getContactId(): int;
    public function getEmail(): int;

    /**
     * @return array<int, string> Associative array, the key is languageId and the value is the localized value.
     */
    public function getLocalizedNames(): array;
    public function getLocalizedDescriptions(): array;
}
```

3. Repositories

### Get entity by ID:
- multi lang entities MUST specify a languageId
- multi shop entities MUST specify a shopId
- multi shop entities which have more complex multi shop behaviour MAY have a getter based on ShopConstraint
- entities which are neither multi shop not multi lang don't need these parameters of course

Two repositories are needed for multi lang interfaces since they don't return the same interface. However for multi shop entities the shopId is always mandatory.

```php
interface ContactRepositoryInterface {
    public function getContact(contactId, languageId, shopId): ContactInterface;
}

interface LocalizedContactRepositoryInterface {
    /**
     * Returns ALL languages present in the database.
     *
     * @param contactId
     * @param shopId
     *
     * @return LocalizedContactInterface
     */
    public function getLocalizedContact(contactId, shopId): LocalizedContactInterface;

    /**
     * @param contactId $
     * @param shopId $
     * @param languages List of filtered languages
     *
     * @return LocalizedContactInterface
     */
    public function getLocalizedContactWithLanguages(contactId, shopId, languages): LocalizedContactInterface;
}
```

When the entity is not found an exception should be thrown `CurrencyNotFoundException` `ContactNotFoundException`

Not found exception MUST extend the associated Domain Exception, but each type of exception MAY implement a generic Exception interface

```php
class CurrencyNotFoundException extends CurrencyException implements EntityNotFoundExceptionInterface {
}
```

### Get byXYZ

```php
interface CurrencyRepositoryInterface {
    public function getCurrency(currencyId, languageId, shopId): CurrencyInterface;
    public function getCurrencyByIsoCode(isoCode, languageId, shopId): CurrencyInterface;
    
    public function getCurrencyBy(array $criteria, languageId, shopId): CurrencyInterface;
}
```

When more than one entity is found (can happen with getByCriteria) an exception should be thrown `NonUniqueCurrencyException` that implements `NonUniqueEntityExceptionInterface`

### List / search / filter (return IDs or full objects)

Decision must be taken between
- return an array of interfaces (we lose the strong type, but no need to implement a collection for each entity, MAYBE more performant?)
- return a collection of interfaces

Decision must be taken between
- findAll
- getAll

```php
interface CurrencyRepositoryInterface {
    /**
     * @return CurrencyInterface[]
     */
    public function findAll(): array;
    public function getAll(): CurrencyCollectionInterface;

    public function findBy(array $filters = [], array $sortBy = [], ?int $limit = null, ?int $offset = null): array;
}
```

4. DBAL queries VS Object models
5. Which rules do we apply with which tools (prevent usage of specific namespaces, keep the namespace clean)
6. Where do we put the implementation

7. Use int or VOs
8. Write operations

### Create

To be decided:
- we'll need hooks to be triggered provided the created interface
- should factory/repository trigger hook or be completely hook independent
- should the hook be dispatched by the handler
- should we have a hook helper which handles automatical hook dispatch + repository call, this helper (seems like a good separation)

```php
interface CurrencyFactoryInterface {
    public function create()
}
```

### Update
### Delete (multishop or not)