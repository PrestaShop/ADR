# 12. Establish rules for form processing

Date: 2020-01-13

## Status

In discussion

## Context

While simplifying forms (https://github.com/PrestaShop/PrestaShop/issues/16482) I noticed that option forms
don't check validation with `form->isValid()` and don't use constraints (https://symfony.com/doc/current/reference/constraints.html).
Also I noticed that option forms were redirected after processing, which would stop constraints from working.


Object forms were successfully using constraints.

In order to make forms consistent and to make constraints usable in all forms I made changes to form processing, so instead of redirecting back to original page form was rendered.

This meant after submitting the form if it redirected you to a different path from original you stayed there.


For example index route of email options page is

    admin_emails_index: /configure/advanced/emails/[GET]

We can't use `POST` to save the form because in email options page there is also list, and list uses admin_emails_index `POST` for list actions.

So for form saving we have

    admin_emails_save_options: /configure/advanced/emails/options[POST]

When you go to admin_emails_index and save the form you get redirected to admin_emails_save_options. With my changes if error occurs you stay
at admin_emails_save_options and constraint errors are correctly displayed.

This introduced a problem with PrestaShop functionalities like multishop which redirect you back to the same page after doing something.
Because admin_emails_save_options is a `POST` route, there is no `GET` route for /configure/advanced/emails/options and after redirection you get an exception: 

https://github.com/PrestaShop/PrestaShop/pull/22471#issuecomment-756738141

After trying to find other solutions conclusion was:

 1) We can't have one action for all forms. (See for more info https://github.com/PrestaShop/PrestaShop/pull/17819)
 2) If action to save the form has different route then rendering it, we can't stay on that route after attempting to process the form
 3) If we redirect it means all info about form is gone

Point 3 also means that constraints cannot be used. They will work and form will be validated correctly using
`form->isValid()`, but because of redirection all data about errors is lost and errors won't be shown to user.
Leading to situations where user won't be able to save the form but won't know why.

So decision was made to not use constraints where form saving route is different from route where it's rendered.

Form can be validated using form data provider (https://devdocs.prestashop.com/1.7/development/architecture/migration-guide/forms/settings-forms/#form-data-providers) and errors can be passed to user using flash messages (https://symfony.com/doc/current/controller.html#flash-messages).
## Rules for form processing:

  1) Each form must have button dedicated to it
  2) Each form should have a separate action dedicated to it
  3) Saving one form should not save other forms
  4) If possible(see 5) form should use constraints to show errors to users
  5) If `POST` action to the same page where form is rendered is not available, form should have a separate route for saving it
and redirect back to the original route. In that case:
   * Constraints cannot be used
   * You must not check for form validity `form->isValid()`