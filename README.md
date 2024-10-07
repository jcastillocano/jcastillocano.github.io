# Juan Carlos Castillo Cano site

## Requirements

1. `gem install bundler`
1. `bundle install` (in case having problems with `ffi` gem, run `bundle
   update`)
1. `bundle exec jekyll serve`

This will serve current site on http://localhost:4000

## Adding content

Add new posts under `_posts` folder (note: media content can be hosted at root folder, but probably a media folder would be ideal). To create a new blog entry just create a new markdown file with this header:

```markdown
---
layout: post
title:  "POST TITLE"
date:   YEAR-MONTH-DAY HOUR:MINUTES:SECONDS +0000
categories: TAG1 TAG2 TAG3 etc
---
```

To reference media content, use this syntax `![Read the docs](/readthedocs.jpg)`

## Author

Juan Carlos Castillo Cano (me)
