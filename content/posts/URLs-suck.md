---
title: URLs are harder than they should be
date: 2023-01-24T16:13:28Z
draft: false
categories:
- Sanic 
---
My current toy web-app is a webmail client. It provides the standard inbox view and then let’s you drill down to view actual mails. I also then have actions you can perform on these. 

The URLs for these are fairly basic: 

- `/mailbox/<ID>`
- `/mailbox/<ID>/<mail ID>`
- `/mailbox/<ID>/<mail ID>/delete`

So far, so whatever. The pain comes with how to construct the drill down urls from the current page. I can get the current URL in my templates with `request.url` and then append `/whatever`. This works until I have a link with a trailing slash. Then my urls end up with double slashes and Sanic responds with a 404. 

By default, Sanic will treat urls with and without trailing slashes as the same URL, so one option would be to use the `strict_slashes` option and prevent the trailing version from working. But then you get extra errors to deal with or redirects to code. 

The answer is to punt the URL construction to the router. Use `app.url_for` or `request.url_for` and have it create the full URL for you. Seems obvious but took me longer than it should have. 

