# Tabokie's Page

Personal site. A remake of [Xe's website](https://christine.website/) with jekyll.

Run `bundle install` followed by `bundle exec jekyll serve` to serve locally.

## Memo

Front matter for normal blog post (named `/_posts/yyyy-mm-dd-name.md`):
```
---
layout: blog
title: Old Blog
category: admin
---
```

Tentative catogorization for blogposts:

- tech: skills and facts only
- review: single, concrete target, e.g. paper, film, device
- fiction
- non-fiction

Front matter for topic post (named `/_(sub)topics/uuid.md`):
```
---
layout: blog
title: Master Topic
childs:
  - /subtopics/uuid
---
```

Referencing blog post:

```
[A Related Post](/tag/yyyy/mm/dd/name.html)
```
