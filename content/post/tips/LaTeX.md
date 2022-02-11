---
title: "LaTeX tips"
date: 2019-10-21
tags: ["LaTeX","Beamer"]
---

## Beamer

### 在Beamer中插入图片

(可参见Beamer文档12.6节)

figure 环境示例：
```tex
\begin{figure}
\centering
\includegraphics[width=\textwidth]{pic/F1.png}
\caption{图中每个方块代表交易，右下角的数字表示交易本身的权重，粗体字表示累积权重。}
\label{fig:bef}
\end{figure}
```

为了使图片的caption有数字标号，在导言区加入下述内容：

```tex
\setbeamertemplate{caption}[numbered]
```

### 从markdown编译Beamer presentation

```
pandoc talk.md -t beamer -o talk.pdf --pdf-engine=xelatex -V mainfont=思源宋体
```
