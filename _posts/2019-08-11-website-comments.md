---
layout: post
title: "Website comments"
date: 2019-08-11
post_id: 9
---

Today I am working on adding functionality on my website for users to add comments. [some people](https://github.com/apps/utterances) have [already done this](https://artsy.github.io/blog/2017/07/15/Comments-are-on/), but I decided to not go with their way and just use the bare-bones github issues functionality without much interfacing. 

Unlike most previous posts, I am writing this post while actually working on the project (instead of after I'm finished.). Hopefully this goes well.

The first thing I just did was add a piece of metadata to each existing post. I called it `comment_id` . I started manually adding it to each of my posts, but then I thought, why am I doing this manually with VIM in a linux terminal? I should be able to at least automate part of it. So, I did some research into using SED to do what I want. Before adding `comment_id`, my post headers looked like this:

```
---
layout: post
title "Website comments"
date: 2019-08-11
---
```

What I wanted to do was to reguar-expression-match on date and insert a line below, like so:

```
---                   
layout: post
title: "Website comments"
daet: 2019-08-11
comment_id: 9
---
```

After looking around [here](https://stackoverflow.com/questions/4510813/sed-regular-expression-over-multiple-lines), I sort of got an idea of what I wanted to do:


```sed
sed -i.bak 'N;/date: [0-9-]\+.*\n/s/\(date: [0-9-]\+\)/\1\ncomment_id: 8/;P;D' file.md
```

The key parts are:

1. `N;/date: [0-9-]\+.*\n/` : Append (`N`) the next line for operationsif the current line matches `date: [0-9-]\+.*\n`
1. `s/\(date: [0-9-]\+\)/\1\ncomment_id: 8/;` : Replace occurrences of `\(date: [0-9-]\+\)` with `\1\ncomment_id: 8` 
1. `P;D` : print everything up to the first new line in pattern space; delete everything up to first new line in pattern space
1. `-i.bak`, `file.md` : do this on `file.md` and create backups with `.bak` suffix 

That ended up working great. I definitely could have automated the numbering process even more with a bash FOR loop or some sed-awk stuff I think, but didn't want to go that far.  I eventually decided to use `post_id` instead of `comment_id`, so I ran `sed -i.bak s/comment_id/post_id/ *.md` on my files (NOTE: If I ever do this again, I will have to be more specific in my expression, since I reference `comment_id` and `post_id` several times in this post).

Once I had done that, I needed to go and manually make an issue for every website post so far. Hopefully this is something I can automate with a github bot in the future.


