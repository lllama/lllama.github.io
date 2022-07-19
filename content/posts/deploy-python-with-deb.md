---
title: "Deploy Python With Deb"
date: 2022-07-19T12:28:33+01:00
draft: false
---
One of the worst parts about working with python is how to share your code, or
deploy it on another machine. If you're following preferred practices, then
you're probably developing using a virtual environment. But that means you need
a way to have the same environment where your app is going to run. Either your
users need to go through several steps to create a virtual environment and then
install your app and its dependencies to it, or you need to ship the
environment along with your app. Or you start having to use containers.

For most python web apps, there is little or no advantage to kubernetes (or
Docker Swarm). You want to run your app using something like gunicorn, uvicorn,
hypercorn, and possibly have these running behind nginx or apache. You probably also want a database and some other services to be running as well. Your deployment environment may well be a single VM that you've created just for this purpose, so having to get kubernetes up and running seems redundant.

Assuming you're running something like Ubuntu, or CentOS, then why can't you
use your package manager to install your app for you? Inspired by [this blog
post by nylas](https://www.nylas.com/blog/packaging-deploying-python/), I
decided to have a look at whether this would be possible for an internal web
app that I've developed.

# The application to deploy

The app has a simple web interface that talks to a DB,
and a couple more system services that need to be running as well. The DB is
postgresql, as that's my DB of choice.

For the web app, I'm using [Sanic](https://sanic.dev/). The framework is very
developer friendly, it supports async for handling large amounts of traffic,
and it comes with its own high performance web server (including [experimental
http/3 support](https://github.com/sanic-org/sanic/releases/tag/v22.6.0). While
the performance of the web server is a good feature, it's the fact that it
simplifies deployment that really helps.

# The deployment plan

The aim is to create a `deb` file that will contain the app, its virtual
environment, and all other assets. The `deb` will specify other packages the
app depends on (e.g. postgresql). Installing it will also install appropriate
`systemd` units that launch the app and allow it to be controlled with
`systemctl`. Logs will ship to `journald`. We will also create new users for
the app to run under, and make sure that we're running with least privileges.
Installing the app will also create the database it will use in postgres, and
create all necessary tables and relations. Future upgrades will allow us to run
schema migrations, either as our own SQL, or using our own python script.
