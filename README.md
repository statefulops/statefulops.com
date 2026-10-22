# statefulops.com

Hugo site for Statefulops, a database consultancy. It uses a small custom theme in `layouts/` with no external theme or submodules.

## Requirements

- [Hugo extended](https://gohugo.io/installation/) v0.146 or newer (built with v0.167)

## Common tasks

```sh
hugo server -D                      # local preview with drafts at http://localhost:1313
hugo new content blog/my-post.md    # new blog post (starts as a draft)
hugo --gc --minify                  # production build into ./public
```

## Where things live

| What | Where |
| --- | --- |
| Site settings, menu, author bio, contact/social links | `hugo.toml` |
| Home page headline | `content/_index.md` |
| Blog posts | `content/blog/` |
| About / Services / Contact pages | `content/*.md` |
| Service cards (home + services page) | `data/services.yaml` |
| Logo and profile photo | `assets/images/` (resized automatically) |
| Styles | `assets/css/main.css` (colours are CSS variables at the top) |
| Syntax highlighting theme | `assets/css/syntax.css` |

## Writing posts

Front matter supports `title`, `date`, `description` (used for cards and SEO), `tags`, `draft`, `toc: false` (hides the table of contents, which appears automatically on posts over 600 words) and `images` (social share image).

Put `<!--more-->` after the intro to control the summary. For callouts:

```md
{{< callout type="warning" title="Careful" >}}
Test this on staging first.
{{< /callout >}}
```

Types: `note`, `tip`, `warning`.
