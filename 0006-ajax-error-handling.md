# 6. Ajax error handling

Date: 2020-03-23

## Status

In discussion

## Context

We need to define a way to handle Ajax calls errors.

As of today, Ajax calls from javascript are not handled always in the same way, sometimes even not handled at all.
When handled, [growl](http://ksylvest.github.io/jquery-growl/) is often used this way:

```javascript
if (response.responseJSON && response.responseJSON.message) {
    $.growl.error({message: response.responseJSON.message});
}
```

This will be working if we have planned to return a json response on the php side but will not work for any other error.