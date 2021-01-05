# 11. Use constants for configuration variables

Date: 2020-01-05

## Status

In discussion

## Context

Configuration names in PrestaShop currently are just hardcoded. It makes it harder to figure out 
which configurations are available, also leaves possibility for mistyping configuration name without noticing.

## Decision

Use constants in Configuration class to define all the configuration names.
 
### Why use Configuration class
It's already exists for all purposes concerning usage of configurations and creation of the new class just to store
constants is not worthwhile in this case.

## First implementation

https://github.com/PrestaShop/PrestaShop/pull/22672

## Consequences

What becomes easier :

- You can't miss-write configuration name
- Using constants uses less memory than plain string
- You can change the name of constant without considering it as breaking change
- It's easier to refactor
- When all configuration names are in one place, it's easier to fine one you are looking for
- Allows deprecating configurations
