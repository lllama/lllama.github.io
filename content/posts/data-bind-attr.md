---
title: "Let's Write a Datastar Plugin"
date: 2025-11-12T22:31:41Z
draft: true
---

Let's build a [datastar](https://data-star.dev) plugin. All datastar
functionality is implemented as plugins. The new RC6 release included a revised
plugin API, so we'll use this. The API is not (yet) documented, but we can still
use it to build our plugin. The plan is to update this document when I spot any
mistakes, so take all of this with a pinch of salt.

## Data Bind Attr

Our new plugin will combine the functionality of two of the core plugins:
`data-bind` and `data-attr`. The `data-bind` plugin sets up a two-way binding
between an element's value and a datastar signal. When used on an input element,
and changes made by the user will be reflected in the signal, and changes to the
signal will be reflected in the element's value.

The `data-attr` plugin sets up a one-way binding between an element's attribute
and a datastar signal. When used on an element, the defined signal's value will
be reflected in the defined attribute.

For our `data-bind-attr` plugin, we want a two-way binding between an arbitrary
attribute and a datastar signal. When used on an element, the defined signal's
value will be reflected in the defined attribute and any changes to the
attribute will be reflected in the signal.

Why is this useful? Web components. Most web component libraries can define
properties which are populated from attribute values. Changes to the properties
will also update the attribute value. So our plugin will let us drive web
components via datastar signals but also allow the web component to update the
signal.

### The Plugin API

Let's have a look at some existing plugins to get an idea of how the plugin API
works. We'll use the `data-attr` plugin as an example.

```typescript
import { attribute } from '@engine'
import { effect } from '@engine/signals'

attribute({
  name: 'attr',
  requirement: { value: 'must' },
  returnsValue: true,
  apply({ el, key, rx }) {
  ...
```

Here we see that we're importing `attribute`. Datastar has attribute, action and watcher plugins. Attributes are put on elements in `data-*` attributes. Actions let us do things like issue requests. And watcher plugins will react to events.

We then see various config settings being passed to `attribute`, along with the
`apply` function. Let's grep all the attribute config options and see what we can use.


 * `name`
 * `requirement`
 * `returnsValue`
 * `argNames`

#### name

Simple enough: the name of the plugin.

#### argNames

The names of arguments made available to the plugin. For example, with the
`data-on` plugin, `argNames` includes `evt` which represents the event object.
The plugin can then use `evt.target` etc.

#### requirement

The `requirement` argument is one of the following values:

 * `allowed` - The attribute must be present on the element.
 * `must` - The attribute must be present on the element.
 * `denied` - The attribute is optional.
 * `exclusive` - The attribute must not be present on the element.

The argument can also take an object with optional `key` and `value` properties.
One or both of these properties must be present. A `key` is the string that
comes after the colon in the `data-*` attribute. E.g. in `data-on:click`, then
`click` is the key. The `value` is the value of the attribute. Both `key` and
`value` can be set as `must`, `denied`, or `exclusive`. `must` means that a
value must be present. `denied` means no value is allowed. And `exclusive` means
that if a key is set then no value is allowed, or vice versa.

If the requirement is defined as:

```js
requirement: {
  key: 'denied'
}
```

Datastar will throw an error if a key is supplied.

```js
requirement: {
  key: 'exclusive'
}
```

Datastar will throw an error if a value is supplied (or if no key is defined).

```js
requirement: {
  key: 'denied'
}
```

Datastar will throw an error if a key is supplied.

Rather than defining the `requirement` argument as an object, you can also define it as a string. For example:

```js
requirement: 'denied'
```

In this case, the requirement will apply to both the `key` and `value` properties. So `denied` means neither is allowed, `must` means both are required, and `exclusive` means only one is allowed.


#### returnsValue

Finally we have `returnsValue` which is a boolean indicating whether the plugin
returns a value or not. At a high level, setting it to `false` means that the
plugin will be performing some action or calculations, but datastar does not
need to get involved after the action is complete. More importantly, it also
means that the attribute's value does not need to be treated as reactive.

If you want to use a reactive expression in the attribute's value, then you need
to set `returnsValue` to `true`. This in turn will mean that the context value
passed to your `apply` function will have an `rx` property that can be used to
get the value of the expression.

### Setup

We'll use typescript, so let's set up a new project:

```bash
tsc --init
```

We now have a `tsconfig.json` file.

###
