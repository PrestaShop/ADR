# 17. Our Backward Compatibility Promise

Date: 2022-03-24

## Status

Accepted

## Context

PrestaShop is a development platform used by developers and integrators to create e-commerce sites. Developers can create other software (including modules & themes) to interact with PrestaShop, extend it, or modify its behavior. For that, they rely on software interfaces which are commonly referred to as "APIs". Whenever any of these interfaces change, existing third-party software must be adapted to work with the new interface -- this is called a "backward-incompatible change", "breaking change" ("BC") or "backward compatibility break" ("BC break"). 

Developing a modular and extensible platform like PrestaShop requires a delicate balance between allowing the software to evolve in the optimal way, and keeping its interfaces as they are, for as long as possible, in order to reduce impact on third-party software. For that, it is important to clarify what belongs to the API and must be protected from changes, and what part of the software is private and can be modified freely.

Further reading: [The Pain Points](https://build.prestashop.com/news/prestashop-in-2019-and-beyond-part-2-pain-points/#no-clearly-defined-api)

## Decision

PrestaShop is committed to follow the [SemVer convention](https://semver.org/), which states that backward incompatible changes are only allowed in major versions.

Only exceptional circumstances (e.g. fixing a security issue, critical bug, ...) can allow introducing backward-incompatible changes in minor or patch versions. Whenever this happens, the maintainer team will make sure to provide a clear list of the affected components.

The following lines detail our backward compatibility promise, and describe what changes are allowed or not in minor or patch versions. Further restrictions (or allowances) might be added in the future. 

### Packages

The PrestaShop code base is composed of two different types of packages:

 - **Production packages:** packages that are shipped in the final release build
 - **Development packages:** tools that can be used by third-party developers to lint, test, format or build whatever they want. They are used as npm or composer dependencies.

Our backward compatibility promise only applies to production packages, except for modules which are not part of the public API.

### PHP

#### General purpose

Note: "Symfony" is mentioned below as relating to the Symfony framework and its components, including the service container.

| Type of Change                                                      | Allowed        |
|---------------------------------------------------------------------|----------------|
| Remove an interface/class                                           | No             |
| Remove public or protected method [^1]                              | No             |
| Change a Symfony service name                                       | No             |
| Remove a Symfony service                                            | No             |
| Change a Hook name                                                  | No             |
| Remove hook parameters                                              | No             |
| Add hook parameters                                                 | Yes            |
| Change value of a constant                                          | Yes            |
| Remove or rename constants                                          | No             |
| Adding argument to an Symfony event                                 | Yes            |
| Modify the type of thrown exceptions (except to a parent class)     | No             |
| Throw a new type of exception from an existing method               | Yes            |
| Add an optional constructor parameter at the end of the args list   | Yes            |

#### Composer dependencies

| Type of Change                                                                     | Allowed |
|------------------------------------------------------------------------------------|----------------|
| Add a production package                                                           | Yes            |
| Update a production package to a version that does not introduce BC breaks         | Yes            |
| Update a production package with a strong impact on userland code and modules [^2] | No             |
| Update/remove a dev package                                                        | Yes            |

#### CQRS

PrestaShop uses Commands and Queries which are part of the public API. Specific rules apply to these objects.

| Type of Change                              | Allowed |
|---------------------------------------------|----------------|
| Change a QueryResult constructor            | Yes            |

#### PHP methods and functions

| Type of Change                                     | Allowed |
|----------------------------------------------------|----------------|
| Rename                                             | No             |
| Modify default parameter values                    | No             |
| Change return type                                 | No             |
| Add parameter                                      | No             |
| Add optional parameter at the end of the args list | Yes            |
| Add strict typing                                  | No             |

#### Symfony service constructors

Since classes whose instantiation is delegated to the Symfony service container are not supposed to be instantiated individually, changing the constructor of those classes is allowed.

#### Symfony Controller actions

Since controllers can be decorated, actions are not in the public API.

#### Experimental Features

Experimental Features and code should be marked with the tags `@internal` or `@experimental` .

Experimental code is not part of the public API.

### Front

#### Back office theme

| Type of Change                                            | Allowed |
|-----------------------------------------------------------|----------------|
| HTML markup                                               | Yes            |
| Modify `id` or `class`                                    | No             |
| Add `id` or `class`                                       | Yes            |
| Add/remove/update npm dev dependencies                    | Yes            |
| Upgrade npm dependencies that do not introduce BC breaks  | Yes            |
| Functions not exposed to the global scope                 | Yes            |
| Functions directly exposed to the global scope            | No             |
| Remove a template file                                    | No             |
| Change template path                                      | No             |
| Change/remove javascript file name [^3]                   | Yes            |


#### Front office

| Type of Change                                            | Allowed |
|-----------------------------------------------------------|----------------|
| Change/remove routes                                      | No             |
| Add routes                                                | Yes            |
| Change Javascript Core functions (name, parameters, return type) | No             |
| Remove variables injected into template engines           | No             |
| Add variables into template engines                       | Yes            |
| Change/remove template file name                          | No             |
| Change/remove javascript file name [^3]                   | Yes            |


#### Theme

| Type of Change                                            | Allowed |
|-----------------------------------------------------------|----------------|
| HTML markup                                               | Yes            |
| Add `id` or `class`                                       | Yes            |
| Add/remove/update npm dev dependencies                    | Yes            |
| Upgrade npm dependencies that do not introduce BC breaks  | Yes            |
| Change/remove functions not exposed to the DOM            | Yes            |
| Change/remove functions directly exposed to the DOM       | No             |
| Change/remove template file name                          | No             |


### Directory structure

All directory names and structure are part of the public API. In the future, a split between public and private directories will be determined.

### Webservices

| Type of Change                     | Allowed |
|------------------------------------|----------------|
| Change/remove routes               | No             |
| Add routes                         | Yes            |
| Remove elements in XML/JSON struct | No             |
| Add elements in XML/JSON struct    | Yes            |
| Remove parameters                  | No             |
| Add optional parameters            | Yes            |
| Add required parameters            | No             |


### Database structure

| Type of Change                              | Allowed |
|---------------------------------------------|----------------|
| Change/remove field type or name to a table | No             |
| Change field default value                  | No             |
| Change/remove table name                    | No             |
| Add new optional field to a table           | Yes            |
| Add a required field                        | No             |
| Change/remove constraints                   | Yes            |


[^1]: Not if the class is tagged as `@internal` or `@experimental`
[^2]: Examples: Doctrine, Guzzle...
[^3]: Only if it's not a production file
