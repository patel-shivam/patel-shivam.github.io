source "https://rubygems.org"

# Local preview toolchain. GitHub Pages builds the site server-side with its
# own Jekyll 3.10; this site uses nothing that differs between the two, and
# Jekyll 4 is what actually installs on current Ruby.
gem "jekyll", "~> 4.4"
gem "kramdown-parser-gfm", "~> 1.1"
gem "webrick", "~> 1.9"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17"
  gem "jekyll-seo-tag", "~> 2.8"
  gem "jekyll-sitemap", "~> 1.4"
end

# Stdlib gems unbundled from recent Ruby releases.
gem "csv"
gem "base64"
gem "bigdecimal"
gem "logger"

gem "tzinfo-data", platforms: [:windows, :jruby]
