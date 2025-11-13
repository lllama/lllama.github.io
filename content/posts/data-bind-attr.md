+++
title = "Let's Write a Datastar Plugin!"
date = 2025-11-12T22:31:41Z
draft = false
+++

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

Here we see that we're importing `attribute`. Datastar has attribute, action and
watcher plugins. Attributes are put on elements in `data-*` attributes. Actions
let us do things within Datastar expressions, like issue network requests or
copying text to clipboard. And watcher plugins will react to events.

We then see various config settings being passed to `attribute`, along with the
`apply` function. Let's grep all the attribute config options and see what we can use.

 * `name`
 * `requirement`
 * `returnsValue`
 * `argNames`

#### name

Simple enough: the name of the plugin.

#### argNames

All plugins are passed an `el` argument that represents the element the plugin
has been used on. The `argNames` parameter contains the names of additional
arguments made available to the plugin. For example, with the `data-on` plugin,
`argNames` includes `evt` which represents the event object. The plugin can then
use `evt.target` etc.

#### requirement

The `requirement` argument is one of the following values, and applies to the
key and/or value defined on the attribute. A `key` is the string that comes
after the colon in the `data-*` attribute. E.g. in `data-on:click`, then `click`
is the key. The `value` is the value of the attribute. E.g. in
`data-text="$foo"`, the value is `$foo`. The `requirement` argument takes one of
the following values:

 * `allowed` - The attribute can be present on the element.
 * `must` - The attribute must be present on the element.
 * `denied` - The attribute is optional.
 * `exclusive` - The attribute must not be present on the element.

The argument can also take an object with optional `key` and `value` properties.
One or both of these properties must be present. Both `key` and `value` can be
set as `must`, `denied`, or `exclusive`. `must` means that a value must be
present. `denied` means no value is allowed. And `exclusive` means that if a key
is set then no value is allowed, or vice versa.

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
  key: 'must'
}
```

Datastar will throw an error if a key is not supplied.

```js
requirement: {
  key: 'allowed'
}
```

A key can be supplied.

Rather than defining the `requirement` argument as an object, you can also define it as a string. For example:

```js
requirement: 'denied'
```

In this case, the requirement will apply to both the `key` and `value`
properties. So `denied` means neither is allowed, `must` means both are
required, and `exclusive` means only one is allowed.

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

#### The `apply` function

The apply function is where you implement the logic of your plugin. It takes a
context object as its argument, which contains information about the attribute
being processed. The context object has the following properties:

  - `el`: HTMLOrSVG              // The element the attribute is on
  - `mods`: Modifiers            // Map of modifier names to their tags
  - `rawKey`: string             // The full raw key (e.g., "bind-attr__debounce.5s")
  - `evt`: Event | undefined     // Event object (if called from an event handler)
  - `error`: ErrorFn             // Error reporting function
  - `key`: string | undefined    // The part after the colon (may be undefined)
  - `value`: string | undefined  // The attribute value
  - `rx`: (...args: any[]) => T | undefined // Reactive function that executes the expression

Let's see how the `data-attr` implements its `apply` function:

```ts
apply({ el, key, rx }) {
```

Here we see that it's using element, the key and the `rx` function. The key
is used to get the name of the attribute that we want to sync with a value. The
`rx` function is used to execute the expression and get the value to sync with
the attribute.

Next, it defines a `syncAttr` function that will be used to update the element's
attribute value:

```ts
const syncAttr = (key: string, val: any) => {
  if (val === '' || val === true) {
    el.setAttribute(key, '')
  } else if (val === false || val == null) {
    el.removeAttribute(key)
  } else if (typeof val === 'string') {
    el.setAttribute(key, val)
  } else {
    el.setAttribute(key, JSON.stringify(val))
  }
}
```

The function uses the `el` element to update the attribute's value, based on the
type of the value passed to the function.

```ts
const update = key
  ? () => {
    observer.disconnect()
    const val = rx() as string
    syncAttr(key, val)
    observer.observe(el, {
      attributeFilter: [key],
    })
  }
  : () => {
    observer.disconnect()
    const obj = rx() as Record<string, any>
    const attributeFilter = Object.keys(obj)
    for (const key of attributeFilter) {
      syncAttr(key, obj[key])
    }
    observer.observe(el, {
      attributeFilter,
    })
  }
```

Next an `update` function is defined depending on whether the user set a key. If
a key was set, then it will be the attribute to update and the return value of
`rx()` will be the value to set. If no key was set, then the value will be an
object where the keys are attribute names, and the values are the attribute
values.

The main thing to note here is that the first thing done will be to disconnect
the observer. Doing so will ensure that any updates to the element do not
trigger further updates and get us stuck in a loop. Once the updates have been
applied, the observer is restarted and set to only watch the attributes that
have been defined.

```ts
const observer = new MutationObserver(update)
const cleanup = effect(update)

return () => {
  observer.disconnect()
  cleanup()
}
```
    
Then we create the `MutationObserver` that will watch our element. We then
create a cleanup function that will do a final sync when the element is
removed from the DOM.
  
## data-bind

The `data-attr` pluign will reflect signal changes into the defined attribues.
What we now need is the reverse: changes to the attributes need to be reflected
into our signals.

Something similar to this is done in the `data-bind` plugin, so let's see how it
does it:

```ts
apply({ el, key, mods, value, error }) {
    const signalName = key != null ? modifyCasing(key, mods) : value
```

We see here that the function uses the `value`, `error` and `mods` properties of
the context passed to it. It then gets the signal name from the key or the value
and applies case modifications as needed.

```ts
let get = (el: any, type: string) =>
   type === 'number' ? +el.value : el.value

 let set = (value: any) => {
   ;(el as HTMLInputElement).value = `${value}`
 }
```

The next part defines a getter and a setter function. It does a simple
conversion to numbers if needed, otherwise just treats the value as a string.

Next it does some checks for what type of element it's on. The getter and setter
are updated to new functions as appropriate. (The bulk of the code is removed
below for brevity.)

```ts

if (el instanceof HTMLInputElement) {
  switch (el.type) {
    case 'range':
    case 'number':
      ...
    case 'checkbox':
      ...
    case 'radio':
      ...
    case 'file': {
      ...
    }
  }
} else if (el instanceof HTMLSelectElement) {
  ...
} else if (el instanceof HTMLTextAreaElement) {
  // default case
} else {
  // web component
  get = (el: Element) =>
    'value' in el ? el.value : el.getAttribute('value')
  set = (value: any) => {
    if ('value' in el) {
      el.value = value
    } else {
      el.setAttribute('value', value)
    }
  }
}
```

Next the plugin deals with the signal's value. The key points are using
`getPath` to get the current value of the signal, and `mergePaths` to ensure
that the signal is present. The `getPath` function is passed a string
representing the signal name. E.g. `getPath('foo')` will get the value of the
`foo` signal. `getPath('foo.bar')` will get the value of the nested signal `bar`
(`{ foo: bar }`).

`mergePaths` takes an array of paths and values to merge into the signal store.
E.g. `mergePaths([['foo.bar', 'quz'], ['foo.baz', 'qum']])` will merge the values
`'quz'` and `'qum'` into the signal store at paths `'foo.bar'` and `'foo.baz'`,
respectively. `mergePaths` also takes an optional `ifMissing` parameter, which
defaults to `false`. If `ifMissing` is `true`, `mergePaths` will create any
missing paths in the signal store. If `ifMissing` is `false`, `mergePaths` will
also delete any paths that are set to `null`. Essentially, `ifMissing: true` means
"only merge in values for properties that don't already exist" - it skips
overwriting anything that's already there.

```ts
const initialValue = getPath(signalName)
const type = typeof initialValue

let path = signalName
if (
  Array.isArray(initialValue) &&
  !(el instanceof HTMLSelectElement && el.multiple)
) {
  const signalNameKebab = key ? key : value!
  const inputs = document.querySelectorAll(
    ...
  ) as NodeListOf<HTMLInputElement>

  const paths: Paths = []
  let i = 0
  for (const input of inputs) {
    paths.push([`${path}.${i}`, get(input, 'none')])

    if (el === input) {
      break
    }
    i++
  }
  mergePaths(paths, { ifMissing: true })
  path = `${path}.${i}`
} else {
  mergePaths([[path, get(el, type)]], {
    ifMissing: true,
  })
}
```

Finally, the plugin creates a `syncSignal` function that gets the value of the
element and updates the signal. The function is added to the `input` and
`change` events, and a cleanup function is returned that will update the signal
with the final value of the element, as well as removing the event listeners.

```ts
const syncSignal = () => {
const signalValue = getPath(path)
if (signalValue != null) {
  const value = get(el, typeof signalValue)
    if (value !== empty) {
      mergePaths([[path, value]])
    }
  }
}

el.addEventListener('input', syncSignal)
el.addEventListener('change', syncSignal)
const cleanup = effect(() => {
  set(getPath(path))
})

return () => {
  cleanup()
  el.removeEventListener('input', syncSignal)
  el.removeEventListener('change', syncSignal)
}
```

### Implementing our plugin

With all these pieces, we should be able to implement our own `bind-attr`
plugin. It should be used as follows:

```
data-bind-attr:attributeName="signal-name-or-path"
```

For our usage, we'll want the `key` to be the attribute name and the value to be
a signal name or path. We set the `requirement` to `'must'` as we want both a
key and a value. In our case, we _don't_ set `returnsValue` to `true` because we
won't be using the `rx()` function. Rather, we'll be accessing the signals
directly with `getPath()` and `mergePaths()`.

Ideally, we'd then use `modifyCasing` to convert the key using the standard
casing modifiers. Sadly, `modifyCasing` is not part of the public plugin API, so
we can't do this for now.

Our plugin will also need the following pieces of code:

 * A simple getter function for getting the attribute value.
 * Code to make sure that the signal exists, and create it if it doesn't.
 * A `syncSignal` function that reads the value of the attribute and updates the
   signal value.
 * A `syncAttr` function that does the opposite and gets the signal value and
   updates the attribute's value.
 * An `update` function that is used to update the signal value when the
   attribute value changes by handling our observer's subscription.
 * A `MutationObserver` that watches the attributes for changes.
 * Code to use `effect` to register our plugin for signal updates and to run
   `syncAttr` when they occur.
 * A cleanup function that performs a final sync of the attributes as well as
   disconnecting the observer.

Here's the full code:

```ts
import { attribute } from '@engine'
import { effect, getPath, mergePaths } from '@engine/signals'
import { modifyCasing } from '@utils/text'

const empty = Symbol('empty')

attribute({
  name: 'bind-attr',
  requirement: 'must',
  apply({ el, key, mods, value, error }) {
    // Ideally we would apply case modifications.
    // Sadly, `modifyCasing` is not part of the public API.
    // const attributeName = modifyCasing(key, mods)
    const attributeName = key
    
    // Simple getter
    let get = (el: any, type: string) =>
      type === 'number' ? +el.getAttribute(attributeName) : el.getAttribute(attributeName)
    
    // Ensure signal exists and has a value.
    const signalName = value
    const initialValue = getPath(signalName) ?? ''
    
    mergePaths([[signalName, initialValue]], {
      ifMissing: true,
    })
  
    // Sync signal value with updated attribute value
    const syncSignal = () => {
      const signalValue = getPath(signalName)
      if (signalValue != null) {
        const value = get(el, typeof signalValue)
        if (value !== empty) {
          mergePaths([[signalName, value]])
        }
      }
    }
    
    // Sync attribute with updated signal value
    const syncAttr = () => {
      const val = getPath(signalName)
      if (val === '' || val === true) {
        el.setAttribute(attributeName, '')
      } else if (val === false || val == null) {
        el.removeAttribute(attributeName)
      } else if (typeof val === 'string') {
        el.setAttribute(attributeName, val)
      } else {
        el.setAttribute(attributeName, JSON.stringify(val))
      }
    }
    
    // Update function to handle observer's observing.
    const update = () => {
      observer.disconnect()
      syncSignal()
      observer.observe(el, {
        attributeFilter: [attributeName],
      })
    }
    
    // Create the observer, cleanup function, and register our plugin for signal
    // updates.
    const observer = new MutationObserver(update)
    observer.observe(el, {
      attributeFilter:[attributeName],
    })
    const cleanup = effect(syncAttr)
    
    // Return our cleanup function and disconnect the observer.
    return () => {
      observer.disconnect()
      cleanup()
    }
}})
```

## Adding our plugin to our application

Our plugin is currently written in Typescript and is importing functions etc as
if it was part of the main datastar bundle. What we need to do is convert our
typescript to javascript and then import the plugin and register with datastar.

We can use `esbuild` to convert the typescript to javascript:

```bash
npx esbuild <path to bindAttr.ts>
```

We can then create a basic HTML file to test our plugin:

```html
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8" />
		<title>DataStar bind-attr Example</title>
		<script type="importmap">
		{
			"imports": {
				"datastar": "https://cdn.jsdelivr.net/gh/starfederation/datastar@1.0.0-RC.6/bundles/datastar.js",
			}
		}
		</script>
		<script type="module" src="/bindAttr.js"></script>		
	</head>
	<body>
		<input type="range" data-bind="count" min="0" max="100" step="1" value="50" /><span data-text="$count"></span>
		<div data-bind-attr:count="count"></div>
	</body>
</html>
```

Use [Caddy](https://caddyserver.com) or your favorite web server to serve the
HTML and JS file, and then use devtools to inspect the element's attribute as
you adjust the range slider.

Note - if you also include datastar in its own script tag, then you may well
cause the browser to hang.

## Final thoughts

The plugin API is nicely written and simple to use. The existing plugins are a
great set of examples that should give you an idea of what's possible -
something like `@clip` at one end, and the whole of Rocket at the other.
