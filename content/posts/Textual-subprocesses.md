---
title: Textual Subprocesses
date: 2023-05-04T23:14:38+01:00
draft: true
categories:
- textual
- python
- til
---
My latest [Textual](http://textual.textualize.io) app runs a couple of executables as subproccesses. However, running the second one causes the TUI to become very laggy and only some inputs were being processed. It turns out that the subprocess was reading from stdin and therefore pinching some of Textual’s input.

Performance could be restored by setting the subprocess’s `stdin` to `subprocess.PIPE` (or the `asyncio` equivalent).