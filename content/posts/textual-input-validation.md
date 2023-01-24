---
title: Validating Input in Textual
date: 2023-01-24T15:05:09Z
draft: false
categories:
- textual
- textual-patterns
- python
---
[Textual](https://textual.textualize.io) has a number of built in widgets to help capture user input. For text input, there is the standard [`Input`](https://textual.textualize.io/widgets/input/) widget. By default, this will let the user enter any characters that they wish.

Whenever a new value is entered, textual will fire off an `Input.Changed` event, that you can handle with an `on_input_changed` method in your `App` (or wherever). You may then be tempted to use this handler to validate the user input - the `Input` has a reactive variable but there’s no easy way to wire something up to an instance’s methods. So you validate the new value, then grab the `Input` from the DOM and then update the `value` attribute. This sort of fits with how you might approach it in a web app - create an `<input>` element and wire something up to its various handlers.

However, in textual, updating the value from the event handler will trigger another changed event to fire. This isn’t really a problem as your input should now be valid so should pass through the handler without being modified but it can lead to some interesting race conditions if a user starts keyboard mashing, and there’s also the possibilty of a infinite loop if you’re not careful.

## If in doubt: subclass

A pattern in textual that differs from web apps is the use of subclassing. This makes sense given that it’s easier to do this in python verus in HTML. (Does it even makes sense in HTML?) You could do something similar if you were using a frontend framework but that’s not a guarantee.

If you look at the code for the demo app (run it with `python -m textual`) you’ll see that there are many subclasses of common widgets. These seem to have been done for CSS reasons: it’s as easy to create and specify a new class as it is to assign IDs to everything, plus you get some sematic ‘markup’ as a bonus.

In our case, if we subclass the `Input` then we can override the default validation methods. For example, the following widget will only allow the user to enter digits:


```python
class NumbersInput(Input):
    def validate_value(self, value):
        return “”.join(char for char in value if char in "0123456789")          
```

## Textual Patterns

As more apps get written, I’m sure more patterns and best practices will appear but I think ‘you probably want a subclass’ is a good one to begin with.