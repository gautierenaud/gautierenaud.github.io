---
title: tests
fileName: tests
tags:
categories:
date: 
lastMod: 
---
https://gohugo.io/

#Go

framework to build static websites

Can detect changes and reload the new version

Markdown -> static HTML

# Setting up Hugo

  + installation

```bash
sudo apt install hugo
# or latest release
go install -tags extended github.com/gohugoio/hugo@latest
```

  + Create project

```bash
hugo new site soexample
cd soexample
# init as a git repo
git init
# add a theme so hugo can display something (can be just downloaded too)
git submodule add  https://github.com/spf13/hyde.git themes/hyde
# should display page with no posts at http://localhost:1313/
hugo serve
```

  + folder will look like

```
├── archetypes
│   └── default.md
├── assets
├── config.toml
├── content # Here I need to put my posts
│   └── post
│       ├── test.md
│       └── tests.md
├── data
├── layouts
├── public
├── resources
│   └── _gen
│       ├── assets
│       └── images
├── static
└── themes
    └── hyde
        └── ...
```

# Logseq integration

  + Logsec 's plugin https://github.com/sawhney17/logseq-schrodinger to export a note to Hugo's format

    + use plugin to export page (page **needs** to be publishable, e.g. public)

    + put page in the right folder in the hugo project (somewhere in `post`)

    + manually put images in the right folder

      + see Images are not exported in single page mode, but are in a specific folder when publishing all public pages


    + `hugo serve` to preview

    + `hugo` to create the static webpage

      + output in `public` folder

# Notes

  + ## Rendering

    + tags are not really supported -> simple text

    + Markup can change drastically between languages

      + bash

![image.png](/assets/image_1674401889433_0.png)

      + vanilla (no markup specified)

![image.png](/assets/image_1674401919128_0.png)

    + Images are not exported in single page mode, but are in a specific folder when publishing all public pages