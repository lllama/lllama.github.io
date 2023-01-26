---
title: Textual Screen Transitions
date: 2023-01-26T11:49:38Z
draft: true
categories:
- textual
- python
---
[Textual](https://textual.textualize.io) brings a little bit of the web to the terminal. Apps have a DOM that is composed of widgets, and layout and styling is all handled with CSS.

Apps can also be made up of multiple [Screens](https://textual.textualize.io/guide/screens/), which could be thought of as separate URLs or pages within a web-app. With this in mind, I started wondering about whether it would be possible to animate a transition between two different screens.

When I write a Single Page Application (SPA) on the web, I normally reach for [Ember](http://emberjs.com). When I first started using it, it was one of the, if not the only, framework that had its own CLI to manage building and developing. It also makes use of a lot of convensions, which lends itself to a nice addon community.

One of the most popular addons is [Liquid Fire](https://ember-animation.github.io/liquid-fire/). Liquid Fire lets you add animations to elements and to also transition between pages of your app.

I decided to see whether something similar could be done with Textual. When moving between different screens, you either [push](https://textual.textualize.io/guide/screens/#push-screen) a screen onto the stack, or [switch](https://textual.textualize.io/guide/screens/#switch-screen) out the current screen for a new one. My idea was to override these methods of an app and use them to perform an animation. For the animation itself, I decided to try and replicate how Liquid Fire does it and to use the [FLIP technique](https://css-tricks.com/animating-layouts-with-the-flip-technique/). The basic idea is to capture the current screen, then capture the destination screen, and then animate between the two.

Textual is full of lots of very helpful functions and features. One in particular is the ability to take screenshots, and this is what I used to capture the two screens. When an `App` changes screen, we first take a screenshot. In particular, we use the `export_text` method to get the text of the screen. We can also ask for all of the escape codes to be included, which means we’ll get all the colours and formatting. We then change to the new screen and take another screenshot.

Once we have the two copies, we then need to find a way to transition between the two. To do this, we use another Screen of our own. We create the new screen, pass it our two copies, and then tell it what transition to use. Once the screen is created, we kick off an `animate` call that animates a [`reactive`](https://textual.textualize.io/guide/reactivity/) variable on our screen. The transition is driven by this, and once it finishes, we pop our screen off the stack, which will leave us with our destination screen. If the transition worked as expected, the two screens will look identical and the app will be ready for the user to interact with.
