---
title: "Start Hugo"
date: 2021-03-24T12:18:03+08:00
draft: false
---

First install hugo on your system

Choose a folder to run this command
```sh
hugo new site blog
```

`cd` into your site folder,
init a `git` repo,
and add a theme.

In my case I forked the [learn](https://github.com/matcornic/hugo-theme-learn) theme in case I want to modify it.

I then added the original branch as "learn" to track new updates.
```sh
cd blog
git init
git submodule add git@github.com:cld4h/hugo-theme-learn.git themes/learn
cd themes/learn
git remote add -f learn https://github.com/matcornic/hugo-theme-learn.git
git diff learn/master origin/master
```

change the `config.toml` file

```toml
# Change the default theme to be use when building the site with Hugo
theme = "learn"

# For search functionality
[outputs]
home = [ "HTML", "RSS", "JSON"]
```

How this Chapter and this page is added
```sh
hugo new --kind chapter hugo/_index.md
hugo new hugo/start.md
```

Start locally
```
hugo serve -D
```

[Learn theme docs](https://learn.netlify.app/en/)
