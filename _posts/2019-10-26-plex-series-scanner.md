---
layout: post
title: "Plex Series Scanner Adjustments"
date: 2019-10-26
post_id: 11
---

I feel like some of my computer-related problems become so much easier to solve when I realize that everything is just code, and code can be adjusted/fixed.

Case in point: I have been using [Plex Media Server](www.plex.tv) for a while now, and I have sometimes run into the issue where Plex would not correctly parse info (Series name, season number, episode number) out of my video filenames. Although I could resolve the problem by just renaming my video filenames, sometimes you don't want to do that (like if the files are from a torrent and you want to keep seeding them). I searched through google to see if there was a way to add custom search/info through the plex interface... but there was nothing. 

THEN, yesterday, I had the thought to dig through the installed plex media server files and see if there was anything I could tweak. Sure enough, in my `C:\Program Files (x86)\Plex\Plex Media Server\Resources\Plug-ins-f2cae8d6b\Scanners.bundle\Contents\Resources\Series` folder, there was a file called `Plex Series Scanner.py`, which happened to be a python script that did the filename parsing!

At the top of the script, there was a global list of regular expressions used to parse filenames:

```python
episode_regexps = [
    '(?P<show>.*?)[sS](?P<season>[0-9]+)[\._ ]*[eE](?P<ep>[0-9]+)[\._ ]*([- ]?[sS](?P<secondSeason>[0-9]+))?([- ]?[Ee+](?P<secondEp>[0-9]+))?', # S03E04-E05
    '(?P<show>.*?)[sS](?P<season>[0-9]{2})[\._\- ]+(?P<ep>[0-9]+)',                                                            # S03-03
    '(?P<show>.*?)([^0-9]|^)(?P<season>(19[3-9][0-9]|20[0-5][0-9]|[0-9]{1,2}))[Xx](?P<ep>[0-9]+)((-[0-9]+)?[Xx](?P<secondEp>[0-9]+))?',  # 3x03, 3x03-3x04, 3x03x04
    '(.*?)(^|[\._\- ])+(?P<season>sp)(?P<ep>[0-9]{2,3})([\._\- ]|$)+',  # SP01 (Special 01, equivalent to S00E01)
    '(.*?)[^0-9a-z](?P<season>[0-9]{1,2})(?P<ep>[0-9]{2})([\.\-][0-9]+(?P<secondEp>[0-9]{2})([ \-_\.]|$)[\.\-]?)?([^0-9a-z%]|$)' # .602.
  ]
```


