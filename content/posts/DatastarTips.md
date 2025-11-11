---
title: "A collection of Datastar tips and tricks"
date: 2025-11-11T21:00:33
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
