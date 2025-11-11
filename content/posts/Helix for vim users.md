---
title: Helix for vim users
date: 2023-04-17T08:43:21+01:00
draft: false
categories:
- til
- helix
- vim
---

# Update

Once again, it appears that what I really want is to just change my Vim colourscheme, rather than switching to a new editor. Though I'll admit that I'm liking Zed a lot these days.

<hr/>


Periodically, I have a go at using an editor other than vim. This normally results in me realising that I just needed a change of vim colour scheme but I’ve just discovered [Helix](https://helix-editor.com/) which might be just enough like vim for me to stick around. 

My reason for giving it a go is that I want something with easy to use language server support. Neovim would probably be a sensible choice but helix seems to be nicely configured out of the box, so I’m going to try that first. 

So here are some notes about translating vim actions into helix:

- Delete to end of line. This is a slightly annoying one. The idea in Helix is that you select things first and then do the action. The movement `gl` goes to the end of the line but _doesn’t_ select by default. So instead we select to the character before the newline and then delete that. 
  - Vim: `D`
  - Helix: `t<enter>d`


## Python tips

You’ll need to install the [python language server](https://github.com/python-lsp/python-lsp-server) to get python support. I recommend [`pipx`](https://pypa.github.io/pipx/) - it’ll install it into its own virtual environment and put it on your path. Helix should then pick it up. 

I also prefer [`black`](https://black.readthedocs.io/en/stable/index.html) for formatting, which needs to have its LSP plugin installed using 

```pipx inject python-lsp-server python-lsp-black```

That will install the plugin into the virtual environment used by the LSP.