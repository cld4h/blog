---
title: "Simple Shortcode to Insert Raw HTML in Hugo"
date: 2021-04-21T20:30:32+08:00
tags: [rawhtml]
---

[Reference](https://anaulin.org/blog/hugo-raw-html-shortcode/)

创建 `layouts/shortcodes/rawhtml.html` 文件如下

```
<!-- raw html -->
{{.Inner}}
```

可以通过如下方式使用

```
{{< rawhtml >}}
  <p class="speshal-fancy-custom">
    This is <strong>raw HTML</strong>, inside Markdown.
  </p>
{{< /rawhtml >}}
```
