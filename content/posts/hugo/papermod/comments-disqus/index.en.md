---
title: "💡 How to enable the comments feature using Disqus in Hugo PaperMod"
date: 2025-07-01T19:13:55+08:00
draft: false
tags: ["hugo", "papermod", "disqus", "comments"]
---

## Register a Disqus account and create a Site

1. Go to [disqus.com](https://disqus.com/)
2. Click the `Sign up` button in the top-right corner and select `Publishers`
3. Create a [New Site](https://disqus.com/admin/create/); the category can be anything
4. On the `Select Your Website Platform` step, scroll to the bottom and click `I don't see my platform listed, install manually with Universal Code`. Then you can skip the installation steps for now — you'll still be able to change the platform and view the installation instructions later
5. Copy the `Shortname`; you will paste this value into Hugo's `config.yml` later

## Enable Comments in PaperMod

> For related feature settings, refer to PaperMod's [exampleSite](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#comments) and the Hugo [comments documentation](https://gohugo.io/content-management/comments/).

1. Add the following settings to your Hugo `config.yml`

```yaml
# Enable comments feature
params:
  comments: true
# Set Disqus Shortname
services:
  disqus:
    shortname: "<your Disqus shortname here>"
```

Replace `<your Disqus shortname here>` with the shortname you copied from Disqus.

2. Since Hugo comes with a built-in Disqus template, you can simply add the following content to `layouts/partials/comments.html`:
    - if the directory or file doesn't exist, create it manually
    - if you need to override Hugo's built-in template, refer to the [official documentation](https://gohugo.io/templates/embedded/#disqus)

```html
{{ partial "disqus.html" . }}
```