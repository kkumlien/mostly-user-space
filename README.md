# Mostly User Space

Source of <https://kkumlien.github.io/mostly-user-space/>, built with [Zola](https://www.getzola.org/) and deployed to GitHub Pages by `.github/workflows/deploy.yml` on every push to `main`.

## Local preview

```sh
zola serve
```

## Layout

- `content/posts/`: one Markdown file per post, front matter in TOML (`+++`)
- `templates/`: minimal Tera templates, no theme
- `static/style.css`: the only stylesheet

## Licence

Posts (`content/`) are [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Templates, styles and config are [MIT](LICENSE).
