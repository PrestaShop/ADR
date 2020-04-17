# 4. Keep QA and Devs HTML selectors separate

Date: 2020-04-17

## Status

Rejected

## Context

Core developers are using JS maps files to keep all the selectors they need to interact with the BO theme.
QA team is keeping their own set of selectors to interact with during automated tests.
Selectors should be mutualized so that when a dev changes a UI component or update a selector, the changes are reflected in the corresponding JS Map file and automated tests using this selector will keep working as intented without human intervention.


The issue motivating this decision, and any context that influences or constrains the decision.

## Decision

QA team and Core developers will populate and maintain their own set of selectors. There is no clear advantage to use a fusioned set of selectors.

Here are the main arguments against this decision:
* Devs use ES6, QA team use ES5. This means using a transpiler and adding libraries (babel).
* There is no clear conventions in existing JS Map files (names and content architecture vary).
* QA team need some selectors with modifiable input (with strings like %ROW or %COLUMN that must be replaced when used), which mean these types of selectors will be unusable for the Core dev team.
* Not the same needs: QA team need navigation selectors, Core dev team need interactive selectors. There is only a few selectors in common.
* Low risk: there is no evidence of any selector modification that caused automated tests to fail. When a test break, it's mostly because of a relatively big UI revamp, something that couldn't be fixed with just a selector modification anyway.

## Consequences

QA team will continue to maintain their own set of selector inside their page objects.
Dev core team will continue to create and maintain their own set of selector to use for JS logic and interactive elements manipulation.
