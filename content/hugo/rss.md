---
title: "RSS in hugo"
date: 2021-04-20T10:33:50+08:00
draft: false
---

[Official docs Reference](https://gohugo.io/templates/rss/)

I copied the [default RSS template that ships with Hugo](https://github.com/gohugoio/hugo/blob/master/tpl/tplimpl/embedded/templates/_default/rss.xml) and changed line 28.

```
28c28
<     {{ range $pages }}
---
>     {{ range .Site.Pages }}
```

I add the following line to my `layouts/partials/menu-footer.html`

```
<p><a href="{{.Site.BaseURL}}index.xml"><i class="fas fa-rss"></i> RSS</a></p>
```
