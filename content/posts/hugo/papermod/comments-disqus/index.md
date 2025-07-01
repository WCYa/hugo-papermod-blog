---
title: "💡 Hugo PaperMod 如何使用 Disqus 啟用 comments 功能"
date: 2025-07-01T19:13:55+08:00
draft: false
tags: ["hugo", "papermod", "disqus", "comments"]
---

## 註冊 Disqus 帳號並建立 Site

1. 至 [disqus.com](https://disqus.com/)
2. 點右上角 `Sign up` 並選擇 `Publishers`
3. 之後建立一個 [New Site](https://disqus.com/admin/create/)，類別可隨意
4. 平台點選最下面的 `I don't see my platform listed, install manually with Universal Code` 按鈕，安裝步驟可先略過，之後還是可以調整平台跟查看 Installation
5. 取得 `Shortname`，此值之後會貼到 Hugo 的 `config.yml` 裡

## PaperMod 啟用 comments 功能

> 相關 Feature 設定可參考 PaperMod 的 [exampleSite](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#comments)，和 Hugo [comments](https://gohugo.io/content-management/comments/) 文件

1. 加入以下設定到 Hugo 的 `config.yml`

```yaml
# 啟用 comments 功能
params:
  comments: true
# 設定 Disqus Shortname
services:
  disqus:
    shortname: "<你剛剛從 Disqus 複製的 Shortname>"
```

2. 因 Hugo 已附帶 Disqus 的模板，所以可直接將以下內容貼到 `layouts/partials/comments.html`
    - 如目錄跟檔案不存在可自行建立
    - 如需要 override Hugo 模板，參考 [官方文件](https://gohugo.io/templates/embedded/#disqus)


```html
{{ partial "disqus.html" . }}
```