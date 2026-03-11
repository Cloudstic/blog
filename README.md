# The Cloudstic Engineering Blog

Our engineering blog, powered by [Jekyll](https://jekyllrb.com/) and [GitHub Pages](https://pages.github.com/).

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Then open [http://localhost:4000/blog](http://localhost:4000/blog).

## Writing a new post

Create a file in `_posts/` with the naming convention:

```
YYYY-MM-DD-title-of-the-post.md
```

Add front matter at the top:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
categories: your-category
---
```

## Deployment

Push to `main` and GitHub Pages will build and deploy automatically.
Make sure GitHub Pages is enabled in the repo settings (source: branch `main`).
