source "https://rubygems.org"

# Run the site locally with:
#   bundle install
#   bundle exec jekyll serve
#
# This is the same stack the GitHub Actions workflow uses
# (.github/workflows/jekyll.yml), so local builds match production.

gem "jekyll", "~> 4.4"

# _config.yml uses `remote_theme: jekyll/minima@3cdd14d` rather than the
# released minima gem, because the `skin: dark` option only exists on
# minima's master branch. This plugin is what makes remote_theme work.
# Pinned to 0.4.x: 0.5+ depends on the native `openssl` gem, which needs a
# full build toolchain and fails to compile on Windows. 0.4.x uses stdlib OpenSSL.
gem "jekyll-remote-theme", "~> 0.4.3"

# Ruby 3+ no longer ships webrick, which `jekyll serve` needs.
gem "webrick", "~> 1.8"

# minima's layouts call {% seo %} and {% feed_meta %}. Because the theme is
# pulled in via `remote_theme` rather than as a gem, its own dependencies are
# not installed for us — they have to be declared here explicitly.
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17"
  gem "jekyll-seo-tag", "~> 2.8"
end

platforms :windows do
  # Windows lacks zoneinfo files.
  gem "tzinfo", ">= 2"
  gem "tzinfo-data"
  # Directory-watching for `jekyll serve --watch`; 0.1.x won't build on Ruby 3.
  gem "wdm", ">= 0.2"
end
