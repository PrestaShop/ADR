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
The getters for localized values returns only one field which is already translated

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

We also have a Localized interface (used mainly in form edition)
```php
interface LocalizedContactInterface {
    public function getContactId(): int;
    public function getEmail(): int;

    /**
     * @return array<int, string> Associative array, the ley is languageId and the value is the localized value.
     */
    public function getLocalizedNames(): array;
    public function getLocalizedDescriptions(): array;
}
```

3. Repositories

Get entity:
- multi lang entities MUST specify a languageId
- multi shop entities MUST specify a shopId
- multi shop entities which have more complex multi shop behaviour MAY have a getter based on ShopConstraint

```php
interface ContactRepository {
    public function getContact(contactId, languageId, shopId): ContactInterface;
}

interface LocalizedContactRepository {
    public function getLocalizedContact(contactId, shopId): LocalizedContactInterface;
    public function getLocalizedContactByLanguages(contactId, shopId, languages): LocalizedContactInterface;
}
```

4. DBAL queries VS Object models
