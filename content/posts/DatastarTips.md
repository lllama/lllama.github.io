---
title: "A collection of Datastar tips and tricks"
date: 2026-06-10T00:00:00
draft: false
---

Some tips and tricks for [Datastar](https://data-star.dev/).

# Building the project

The project is developed in a private repo and then a public cut of the code is pushed to the public repo. Bundles of the code are also published but there are no instructions on how to actually build it yourself. Until now!

Run the following from within the `library/` directory of the repo:

```bash
npx esbuild --bundle src/bundles/datastar.ts \
            --outdir=../bundles/ \
            --minify --sourcemap \
            --target=es2023 \
            --format=esm
```

That should produce a bundle in the top level `bundles` directory. Repeat with the other files in the `library/src/bundles/` dir for all builds. Use the `--define ALIAS=...` flag to change the alias prefix.


# Web components with Lit

Web components driven by signals are a great way to use Datastar. There are two data-star attributes that are helpful: `data-bind` and `data-attr`.

## `data-bind`

The `data-bind` attribute will bind a datastar signal to a property on the custom element. This means that if you defined a property with the `@property` decorator, then any signal updates will trigger an element update as well. However, there are a couple of gotchas when using this approach:

### Potential race condition

If datastar is loaded and walks the DOM before the custom element is defined, then datastar will bind to the empty element in the DOM. When the custom element is defined and replaces the element in the DOM, then the datastar binding will not be refreshed and so signal updates will not trigger a component update. To avoid this, make sure that your custom element JS bundle is loaded and run before datastar is loaded.

### Binding complex objects

If the signal you wish to bind is an array - _and has been defined before the binding_ - then datastar will assume you want to bind the items in the array to the child elements of your web component. The result will be that your property will only contain the first element in the array, rather than the array itself. The work around is to _not_ define your signal until the binding has happened. Normally this will involve not including your signal in any `data-signals` attributes, and then patching in the array once your `data-bind` has been run.

Also, due to signal nesting/namespacing, you cannot bind an object to a property. E.g. if you try and do `$foo = {'some': 'object', 'bar': 'baz'}` then datastar will treat this as a nested set of signals and `data-bind:foo` or `data-bind="$foo"` will fail with an error in the console.

TL;DR - the new `data-bind` behaviour is great for simple signals but can trip you up if you decide to get too fancy.


## `data-attr`

The safe fallback is to use `data-attr`. With this, you can bind an attribute to a signal value and trigger updates when the signal changes. You won't get tripped up by the array behaviour, but you will still bump into the nested object issue.

It should also avoid the race condition, as datastar will be setting the attribute values, which will not be effected when the custom element is initialised.

## Ignition

One other option is to use [Ignition](https://github.com/dataSPA/ignition), which lets you use `this.dsRoot` to access datastar signals and trigger updates when a signal changes. This will avoid the race condition and the array issue but could still trip you up if you want to access nested objects.