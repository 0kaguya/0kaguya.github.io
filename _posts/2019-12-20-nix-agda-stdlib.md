---
layout: post
title: "Enable Agda's standard library when installing with Nix"
categories: fix
tags: [nix, agda, daily]
date: 2019-12-20
---

Ispired by [Isaac's post](https://blog.ielliott.io/agda-nixos/).

Package `AgdaStdlib` from nixpkg doesn't contain a
[`standard-library.agda-lib`](https://github.com/agda/agda-stdlib/blob/master/standard-library.agda-lib)
file as we need, but we can write one manually.
Since nix links agda's standard library to `share/agda/`, which is `~/.nix-profile/share/agda/`, create
`standard-library.agda-lib` containing follows:

``` 
name: standard-library
include: /home/<username>/.nix-profile/share/agda/
```

Save these to a proper location, for example, `~/.agda/standard-library.agda-lib`, 
then the rest will be almost same with the [instruction](https://github.com/agda/agda-stdlib/blob/master/notes/installation-guide.md).
