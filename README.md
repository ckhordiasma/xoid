# Source Code Repository for Christopher Kodama Personal Website

This repository is published as a website via GitHub Pages here: https://www.xoid.net

## Instructions for running locally

1. Follow the install guide for jekyll [here](https://jekyllrb.com/docs/), which involves installing Ruby, RubyGems, and some other stuff.

2. `cd` into the repo and run `bundle exec jekyll serve` 

## Issues I ran into

### `Cannot load such file -- json`

this meant I didn't have a json gem available, so I installed it with `bundle add json` and then `bundle install`
