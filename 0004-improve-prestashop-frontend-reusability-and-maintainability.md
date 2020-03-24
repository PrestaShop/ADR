# 4. Improve PrestaShop frontend reusability and maintainability

Date: 2020-03-24

## Status

In discussion

## Context

I recently seen that there are a lot of bootstrap component used in the project. Actually on migrated pages, the style and markup is mainly coming from Prestakit.

That's a good point, but we need to improve it a litle bit. Except for some reusables components like Gid and recently [HelpBoxes](https://github.com/PrestaShop/PrestaShop/pull/18184) everytime we use a component, we duplicate the markup.

The main component showing that is Cards. On every migrated page, we totally duplicate the card component's markup.

That's a big problem as if Bootstrap decide to change the markup of that component, we'll be stuck on our version until we work on every pages to change the markup, add classes...

Also, recently we did some improvements and modifications to the UIKit, HelpBoxes needed some markup modifications to work properly, which was hard to do as they where duplicated a lot.

Another painpoint is the use of cards. That component should be mainly used to display short component, listed, like a post on a social network, theme cards, modules cards. Actually they are used on every migrated pages as containers, that's a wrong use. If we had this component reusable, in a single file, included everytime we want to use it, we could easily replace it on the entire BO with a custom component which act like a container.

## Decision

What I would propose is :

Everytime we need to use a component, we should think about some points.

1. Is this component used on many pages ?
2. Would it be important that we can globally change this component on an easy way?

If that component fill these conditions, we should do something like this :

```
{% set $content %}
  {% include 'grids.html' with {'grid': $gridDatas %}
{% endset %}

{% set $footer %}
  <a class="btn btn-primary">
    Save
  </a>
{% endset %}}
```

And use that component like this :

```
{% include '@Common/Card/card.html.twig' with {'title': 'Avoirs', content: $grid, footer: $footer} %}
```

You can take a look at [HelpBoxes](https://github.com/PrestaShop/PrestaShop/pull/18184) PR.

It doesn't stick to Prestakit, we also need to make project components reusable globally. For example, a customer card, actions bar, tabs, helper cards can be reusable, what about creating a file that we can include everywhere ?

## Consequences

First of all, we keep the ability to customize the markup as you need when working with these components by adding every properties you would need :

1. Adding classes
2. Content is inside a variable that you pass to this component
3. If the component require it, manage orders of reusable markups

What becomes easier :

1. Everyone use the exact same markup when developping a new feature.
2. Module developers could be able to reuse these components the exact same way as core does (Not sure?) and that would improve the integrity of the front-end.
3. If Bootstrap change something, we can easily adapt our front-end to upgrade it.
4. Easier ability of the core to modify the UIKit and apply these changes to the backoffice.
5. Easier front-end integration workflow and easier to document it.
6. Easier review as you focus on main feature of the page/feature with a clear markup.

What becomes harder :

1. You can't go yolo adding components markup without thinking reusable as a modern js front-end framework require.
2. You need to check if a component exist, and if it doesn't you need to create it reusable for next needs of future development.
3. If you don't use existing components, you'r review can require changed.
