---
permalink: /development
title: DEVELOPMENT
layout: default
---
# Description

This file explains the process of local development for this repository.

# Dependencies

This repository is built using [Jekyll](https://jekyllrb.com/) along with some plugins and a theme.

To install the necessary dependencies on Ubuntu use:
```
sudo apt-get install ruby-full build-essential zlib1g-dev
gem install jekyll bundler
```
It might be necessary to specify installation destination for your Gems:
```
export GEM_HOME="$HOME/.gems"
export PATH="$GEM_HOME/bin:$PATH"
```
For instructions on other systems use [these docs](https://jekyllrb.com/docs/installation/).

Then you can install other Gems like the `just-the-docs` theme:
```
bundle install
```

# Building

To simply build the site use:
```
bundle exec jekyll build --config _config_local.yml
```
This will generate the site files and put them under `_site`.

But if you want Jekyll to continuously build and also serve the site use:
```
bundle exec jekyll serve --config _config_local.yml --incremental
```
This should make it available under http://127.0.0.1:4000/.
