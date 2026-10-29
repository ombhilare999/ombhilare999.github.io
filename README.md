# omkarbhilare.com

Personal academic site, built with Jekyll and deployed to the `gh-pages` branch by
[.github/workflows/deploy.yml](.github/workflows/deploy.yml).

## Editing

| What | Where |
| --- | --- |
| Bio, section order | `index.html` |
| Name, links, news count | `_config.yml` |
| Publications | `_data/publications.yml` (thumbnails in `assets/img/publication_preview/`) |
| News | `_news/*.md` |
| Projects | `_projects/*.md` |
| Blog posts | `_posts/*.md` |
| Styles | `assets/css/main.css` |

## Local preview

```sh
bundle install
bundle exec jekyll serve
```
