# 13. Use TypeScript on the project

Date: 2021-02-25

## Status

Accepted

## Context

While the whole JavaScript community welcomes TypeScript easily, our project's JS is not typed at all. To improve the maintainability of the project and use the latest TC39 features, a good idea would be to use TypeScript.

Benefits of using it for the project:
- Detect bugs before pushing PRs. TypeScript users say that globally, it allows detecting around 15% of bugs that you would detect by testing.
- Use latest features such as Optional Chaining, Tuples, and Records... really early.
- Types are increasing the quality of the project because we would be able to detect dangerous changes, related bugs... If we use it on the PHP side, why don't we use types while using JS?
- Vue 3 offers a new API: Composition API, this one is pretty easy to use with TypeScript as it's mainly functional programming instead of opinionated APIs of Vue, that would be a good move to preshot the Vue update in the BO.

## Decision

Add the possibility to transpile ts files inside every js folder of the project with webpack.

[Here is a POC](https://github.com/PrestaShop/PrestaShop/pull/23221) - basically using TypeScript on a small part of the PrestaShop Grid system.

## Consequences

It's a bit more difficult if you're not used to developing with types, it will probably require a bit of time for contributors who are not used to work with TS, but almost every known project is refactoring their JS with TS, it's beginning to be a must-have on maintained projects. 

The worst case scenario would be that most JS contributors stop contributing because of a lack of knowledge, but we don't have many JS contributions right now and I doubt this is a big risk.
