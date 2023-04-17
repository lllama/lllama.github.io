---
title: sanic-kit
date: 2023-04-17T20:35:00+01:00
draft: false
categories:
- sanic
- python
- sanic-kit
---

[HTMx](https://htmx.org/) has been a revelation for writing web apps. I’d previously used my preferred DERP stack (Django, Ember, REST framework, postgresql) which produced nice enough apps but took a lot of effort and had  lots of redundant code: Postgres tables needed to be Django models, which needed to be DRF serialisers, which needed Ember Data models, and then finally rendered to the DOM. 

Writing with htmx meant I could have most of the shiny but I only need to produce the html on the server.

htmx’s emphasis on just returning html for me thinking about a simpler way to actually write apps. Having used Django, FastAPI and Starlette, there is a lot of boilerplate required to wire a url up to a template. So I had an attempt at implementing file path routing for Starlette and came up with [Dark Star](https://lllama.github.io/dark-star/). It worked but didn’t feel that ergonomic. 

I was then introduced to [Sanic](https://sanic.dev/). Sanic has a Flask-like syntax and is fully async. It has an extensions module that makes using jinja templates super easy and many other features that make for a brilliant developer experience. One of its biggest wins is that it ships with a production-grade server: the server you dev on is the server you use in production. 

For a different project I was looking at [Svelte](https://svelte.dev/) which led me to [Svelte Kit](https://kit.svelte.dev/). Svelte Kit allows you to write a whole web app in Svelte, and it makes use of file path routing. So I started work on [Sanic-kit](https://github.com/lllama/sanic-kit). It copies Svelte Kit as much as makes sense to and lets you keep your handler code right next to your templates. When you build your app it translates everything to a “proper” Sanic app (that follows the practices from the [Sanic Book](https://sanicbook.com/) as much as possible) and runs it with Sanic’s excellent server. 

It’s early days so far but the approach seems sound so I’ll post more updates here as it progresses.