# 2. Define ACL rules for Symfony pages

Date: 2019-01-22

## Status

Accepted

## Context

During migration, some Symfony controllers were created with the following access rules:
- index (display the page) can be accessed by a user if he is granted READ permission
- form submission requires either CREATE, UPDATE, DELETE permissions (depends on what the form does)

Others were created with the following access rules:
- index (display the page) can be accessed if the user is granted either READ, CREATE, UPDATE or DELETE permissions
- form submission requires either CREATE, UPDATE, DELETE permissions (depends on what the form does)

The 2nd kind of controllers were implementing the rule "if you can modify it, you should be able to display it".

There was a need to decide of a global rule to be applied systematically.

## Decision

We agreed to use the 1st set of rules, meaning:

- index can be accessed if you have READ permission and only this one
- if the page allows to create an item, this action requires CREATE permission
- if the page allows to edit an item, this action requires UPDATE permission
- if the page allows to delete an item, this action requires DELETE permission
- if the page allows to modify prestashop settings, this action can be used by users who are granted either CREATE, UPDATE, DELETE permissions

## Consequences

We need to update all controllers already built so they follow this rule.
We need to make sure controllers that will be added will follow this rule.

See issue https://github.com/PrestaShop/PrestaShop/issues/12270
See specs https://github.com/PrestaShop/prestashop-specs/issues/1

Merchants now need to be accurate when granting access to their employees: if an employee needs to use one of the pages of the BO it must be granted the READ permission even if he already has either CREATE, UPDATE or DELETE permissions.