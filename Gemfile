source "https://rubygems.org"

# 이 사이트는 GitHub Pages 가 자체 Jekyll 로 빌드한다.
# github-pages 젬은 GitHub 이 실제로 사용하는 Jekyll 버전과 플러그인 묶음을
# 그대로 고정해 주므로, 로컬 빌드 결과가 실서비스와 어긋나지 않는다.
# _config.yml 의 plugins 에 적은 jekyll-feed / jekyll-sitemap / jekyll-paginate
# 는 모두 이 젬에 포함되어 있어 따로 선언하지 않는다.
gem "github-pages", group: :jekyll_plugins

# Ruby 3.0 부터 webrick 이 표준 라이브러리에서 분리되어,
# 없으면 `bundle exec jekyll serve` 가 실패한다.
gem "webrick", "~> 1.8"
