# 17. Our Backward Compatibility Promise

Date: 2022-03-24

## Status

N/A

## Context

If you are a theme or module developer, the following guidelines will help you to ensure smooth upgrades to all futur minor releases.
This page describes rules and best practices for backward compatible development.

### Packages

The PrestaShop code base is composed of two different types of packages:

 - **Production packages:** these are packages that are shipped 
 - **Development packages:** these are tools that can be used by third-party developers to lint, test, format or build whatever they want. They are used as npm or composer dependencies.

Backward compatibility guarantees only apply to the production packages except for modules which are not part of the public API.


### PHP

#### General purpose

| Type of Change                                                 | Change Allowed |
|----------------------------------------------------------------|----------------|
| Interface/class removal                                        | No             |
| Public and protected method removal [1]                        | No             |
| Change/remove service name                                     | No             |
| Composer dependencies without BC breaks                        | Yes            |
| Composer dependencies used a lot                               | No             |
| Composer dev dependencies                                      | Yes            |
| Change Hook name                                               | No             |
| Remove hook parameters                                         | No             |
| Add hook parameters                                            | Yes            |
| Change value of a constant                                     | Yes            |
| Remove or rename constants                                     | No             |
| Adding argument to an event                                    | Yes            |
| Modifying the types of thrown exceptions                       | No             |
| Throw a new type of exception from an existing method          | Yes            |
| Adding a optional constructor parameter at the end of the list | Yes            |
|                                                                |                |

#### CQRS

| Type of Change            | Change Allowed |
|---------------------------|----------------|
| Queries                   | No             |
| QueryResult (constructor) | yes            |
| QueryResult (getters)     | No             |
|                           |                |

#### Methods and functions

| Type of Change           | Change Allowed |
|--------------------------|----------------|
| Rename                   | No             |
| Default parameter values | No             |
| Return type              | No             |
| Add parameter            | No             |
| Force strict typing      | No             |
|                          |                |

#### Symfony Controller actions

Since controllers can be decorated, actions are not in the public API.

#### Experimental Features

Experimental Features and code should be marked with the `@internal` tags.

#### Security issues

Note that backward compatibility breaks are tolerated if they are required to fix a security issue.

### Front

#### Back office

| Type of Change                         | Change Allowed |
|----------------------------------------|----------------|
| HTML markup                            | Yes            |
| `id` and `class`                       | No             |
| Npm dev dependencies                   | Yes            |
| Npm dependencies                       | No             |
| Functions not exposed to the DOM       | Yes            |
| Functions directly exposed to the DOM  | No             |
| Change/remove template file name       | No             |
| Change/remove javascript file name [2] | Yes            |
|                                        |                |

#### Front office

| Type of Change                                            | Change Allowed |
|-----------------------------------------------------------|----------------|
| Change/remove routes                                      | No             |
| Add routes                                                | Yes            |
| Javascript Core functions (name, parameters, return type) | No             |
| Remove injected variables into template engines           | No             |
| Add variables into template engines                       | Yes            |
| Change/remove template file name                          | No             |
| Change/remove javascript file name [2]                    | Yes            |
|                                                           |                |

#### Default theme

| Type of Change                        | Change Allowed |
|---------------------------------------|----------------|
| HTML markup                           | Yes            |
| `id` and `class`                      | No             |
| Npm dev dependencies                  | Yes            |
| Npm dependencies                      | No             |
| Functions not exposed to the DOM      | Yes            |
| Functions directly exposed to the DOM | No             |
| Change/remove template file name      | No             |
|                                       |                |

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
| Add parameters                     | Yes            |
|                                    |                |


### Database structure

| Type of Change                              | Change Allowed |
|---------------------------------------------|----------------|
| Change/remove field type or name to a table | No             |
| Change field default value                  | No             |
| Change/remove table name                    | No             |
| Add new optional field to a table           | Yes            |
| Introducting a required filed               | No             |
| Change/remove constraints                   | Yes            |
|                                             |                |


[1] Not if the class is tagged as `@internal`
[2] Only if it's not a production file
