# 17. Our Backward Compatibility Promise

Date: 2022-03-24

## Status

N/A

## Context

The following lines describe what changes are considered backward compatible and what changes are not.

PrestaShop attempts to follow [SemVer](https://semver.org/) which states that not-backward compatible changes are only allowed in major versions.

Some circumstances might require to break this promise. Whenever such a usecase happens, the maintainer team will make sure to provide a clear list of the not backward compatible changes introduced in a patch or minor version.

For example backward compatibility breaks are allowed if they are required to fix a security issue.

### Packages

The PrestaShop code base is composed of two different types of packages:

 - **Production packages:** these are packages that are shipped in the final release build
 - **Development packages:** these are tools that can be used by third-party developers to lint, test, format or build whatever they want. They are used as npm or composer dependencies.

Backward compatibility guarantees only apply to the production packages except for modules which are not part of the public API.

### PHP

#### General purpose

Note: Symfony is mentioned below as rely on Symfony framework and its container

| Type of Change                                                      | Change Allowed |
|---------------------------------------------------------------------|----------------|
| Interface/class removal                                             | No             |
| Public and protected method removal [1]                             | No             |
| Change Symfony service name                                         | No             |
| Remove Symfony service                                              | No             |
| Change Hook name                                                    | No             |
| Remove hook parameters                                              | No             |
| Add hook parameters                                                 | Yes            |
| Change value of a constant                                          | Yes            |
| Remove or rename constants                                          | No             |
| Adding argument to an Symfony event                                 | Yes            |
| Modifying the type of thrown exceptions                             | No             |
| Throw a new type of exception from an existing method               | Yes            |
| Adding a optional constructor parameter at the end of the args list | Yes            |
|                                                                     |                |

#### Composer dependencies

| Type of Change                                                              | Change Allowed |
|-----------------------------------------------------------------------------|----------------|
| Upgrade composer dependencies that do not introduce BC breaks               | Yes            |
| Composer dependencies with a strong impact on userland code and modules [2] | No             |
| Updating/removing composer dev dependencies                                 | Yes            |

#### CQRS

PrestaShop uses Commands and Queries. Specific rules apply to these PHP objects that define a contract.

| Type of Change                              | Change Allowed |
|---------------------------------------------|----------------|
| Change a QueryResult constructor            | Yes            |
|                                             |                |

#### PHP methods and functions

| Type of Change                                     | Change Allowed |
|----------------------------------------------------|----------------|
| Rename                                             | No             |
| Default parameter values                           | No             |
| Return type                                        | No             |
| Add parameter                                      | No             |
| Add optional parameter at the end of the args list | Yes            |
| Add strict typing                                  | No             |
|                                                    |                |


#### Symfony services constructors

As classes whose instantiation is delegated to the Symfony container are not supposed to be instantiated otherwise, changing the constructor of those classes is allowed.

#### Symfony Controller actions

Since controllers can be decorated, actions are not in the public API.

#### Experimental Features

Experimental Features and code should be marked with the tags `@internal` or `@experimental` .

Experimental code is not part of the public API.

### Front

#### Back office theme

| Type of Change                                            | Change Allowed |
|-----------------------------------------------------------|----------------|
| HTML markup                                               | Yes            |
| `id` and `class`                                          | No             |
| Npm dev dependencies                                      | Yes            |
| Upgrade npm dependencies that do not introduce BC breaks  | Yes            |
| Functions not exposed to the global scope                 | Yes            |
| Functions directly exposed to the global scope            | No             |
| Remove a template file                                    | No             |
| Change template path                                      | No             |
| Change/remove javascript file name [3]                    | Yes            |
|                                                           |                |

#### Front office

| Type of Change                                            | Change Allowed |
|-----------------------------------------------------------|----------------|
| Change/remove routes                                      | No             |
| Add routes                                                | Yes            |
| Javascript Core functions (name, parameters, return type) | No             |
| Remove injected variables into template engines           | No             |
| Add variables into template engines                       | Yes            |
| Change/remove template file name                          | No             |
| Change/remove javascript file name [3]                    | Yes            |
|                                                           |                |

#### Theme

| Type of Change                                            | Change Allowed |
|-----------------------------------------------------------|----------------|
| HTML markup                                               | Yes            |
| `id` and `class`                                          | No             |
| Npm dev dependencies                                      | Yes            |
| UUpgrade npm dependencies that do not introduce BC breaks | Yes            |
| Functions not exposed to the DOM                          | Yes            |
| Functions directly exposed to the DOM                     | No             |
| Change/remove template file name                          | No             |
|                                                           |                |

### Directory structure

All directories name and structure are part of the public API until the split between public and private directories is done.

### Webservices

| Type of Change                     | Change Allowed |
|------------------------------------|----------------|
| Change/remove routes               | No             |
| Add routes                         | Yes            |
| Remove elements in XML/JSON struct | No             |
| Add elements in XML/JSON struct    | Yes            |
| Remove parameters                  | No             |
| Add optional parameters            | Yes            |
| Add required parameters            | No             |
|                                    |                |


### Database structure

| Type of Change                              | Change Allowed |
|---------------------------------------------|----------------|
| Change/remove field type or name to a table | No             |
| Change field default value                  | No             |
| Change/remove table name                    | No             |
| Add new optional field to a table           | Yes            |
| Introducting a required field               | No             |
| Change/remove constraints                   | Yes            |
|                                             |                |


[1] Not if the class is tagged as `@internal` or `@experimental`
[2] Examples: Doctrine, Guzzle...
[3] Only if it's not a production file
