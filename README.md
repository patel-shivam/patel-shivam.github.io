# patel-shivam.github.io

Personal academic website — <https://patel-shivam.github.io>

Built with [Jekyll](https://jekyllrb.com) and served by GitHub Pages. No external
theme: the layouts and stylesheet in this repo are the whole design.

## Editing

| What you want to change | File |
| --- | --- |
| Bio, experience, awards | `index.md` |
| News items | `_data/news.yml` |
| Papers | `_data/publications.yml` |
| CV PDF | `files/CV_current.pdf` |
| Colours, type, spacing | `assets/css/style.css` |
| Nav links, site metadata | `_config.yml` |

Adding a paper means adding one entry to `_data/publications.yml`; set
`selected: true` to also surface it on the homepage.

## Running locally

```sh
bundle install
bundle exec jekyll serve --livereload   # http://127.0.0.1:4000
```

Requires Homebrew Ruby (`brew install ruby`); macOS system Ruby is too old.
`~/.zshrc` puts it on PATH.

If a native gem ever fails with `'iostream' file not found`, macOS has a stale
`/Library/Developer/CommandLineTools/usr/include/c++/v1` shadowing the SDK's real
libc++ headers. Clang searches it first, finds it nearly empty, and never falls
through. Move it aside:

```sh
sudo mv /Library/Developer/CommandLineTools/usr/include/c++ \
        /Library/Developer/CommandLineTools/usr/include/c++.bak
```

## Deploying

Push to `main`. GitHub Pages builds the site itself with Jekyll 3.10 and ignores
the `Gemfile` above, which exists only for local previews; this site uses nothing
that differs between Jekyll 3 and 4. All three plugins are on the GitHub Pages
allowlist, so no Actions workflow is needed.
