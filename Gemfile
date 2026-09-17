source "https://rubygems.org"

gem "jekyll", "~> 4.4"

# liquid < 4.0.4 calls String#untaint, which Ruby 3.2+ removed.
gem "liquid", ">= 4.0.4"

# Theme plugins (jekyll-theme-serial-programmer)
group :jekyll_plugins do
  gem "jemoji"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-feed"
end

# Ruby 3.4+ moved these out of the default gems; Jekyll still needs them.
gem "csv"
gem "base64"
gem "bigdecimal"
gem "webrick"

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]
