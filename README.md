# Academic Homepage


## Installation

```bash
brew install rbenv ruby-build imagemagick
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
omz reload
rbenv install 3.1.2
rbenv global 3.1.2
omz reload
gem install bundler
bundle install
bundle exec jekyll serve
```
