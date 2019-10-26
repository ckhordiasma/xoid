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
and later in the code (within a for loop iterating over all files) there was the following:
```python
        for rx in episode_regexps[0:-1]:
          match = re.search(rx, file, re.IGNORECASE)
          if match:

            # Extract data.
            show = match.group('show') if match.groupdict().has_key('show') else ''
            season = match.group('season')
            if season.lower() == 'sp':
              season = 0
            episode = int(match.group('ep'))
            endEpisode = episode
            if match.groupdict().has_key('secondEp') and match.group('secondEp'):
              endEpisode = int(match.group('secondEp'))
```

So really, all I had to do was to add an entry to the initial regexp list in order to make my files work! Pretty easy! 

From the installed plex files, it seemed like python2.7 was being used. I made a short script to test out my new regex to see if it worked:

```python
import os.path, re

episode_regexps = [

    '\[.*?\](?P<show>.*?)(?P<season>[0-9])[\._\- ]+(?P<ep>[0-9]+)',    # MY NEW REGEX
    '(?P<show>.*?)[sS](?P<season>[0-9]+)[\._ ]*[eE](?P<ep>[0-9]+)[\._ ]*([- ]?[sS](?P<secondSeason>[0-9]+))?([- ]?[Ee+](?P<secondEp>[0-9]+))?', # S03E04-E05
    '(?P<show>.*?)[sS](?P<season>[0-9]{2})[\._\- ]+(?P<ep>[0-9]+)',                                                            # S03-03
    '(?P<show>.*?)([^0-9]|^)(?P<season>(19[3-9][0-9]|20[0-5][0-9]|[0-9]{1,2}))[Xx](?P<ep>[0-9]+)((-[0-9]+)?[Xx](?P<secondEp>[0-9]+))?',  # 3x03, 3x03-3x04, 3x03x04
    '(.*?)(^|[\._\- ])+(?P<season>sp)(?P<ep>[0-9]{2,3})([\._\- ]|$)+',  # SP01 (Special 01, equivalent to S00E01)
    '(.*?)[^0-9a-z](?P<season>[0-9]{1,2})(?P<ep>[0-9]{2})([\.\-][0-9]+(?P<secondEp>[0-9]{2})([ \-_\.]|$)[\.\-]?)?([^0-9a-z%]|$)' # .602.
  ]

fname = '[NeSubs] Karakai Jouzu no Takagi-san 2 - 02 [English Softsub].mkv'

for rx in episode_regexps[0:-1]:
    print(rx)
    match = re.search(rx,fname, re.IGNORECASE)
    if(match):
        print("match")
        show = match.group('show') if match.groupdict().has_key('show') else ''
        season = match.group('season')
        episode = int(match.group('ep'))
        print 'show: ', show
        print 'season: ', season
        print 'episode: ', episode
```

The first issue I noticed was that the loop only covered `episode_regexps[0:-1]`, which meant that the last list element in `episode_regexps` was not included in the loop! I don't know if that was intentional or a bug, but I removed the `[0:-1]` from the script to make it loop over everything. 

The filenames in my test code had this format: `[<sub group>] <Series name> <season num> - <ep num> <other info>.mkv`, and I built my regex to pull out the series name, season num, and episode number. I don't want to go into regexes at all, but the main point is that it actually worked, and when I made that modification and checked plex, it was able to correctly load in the files and their metadata! 

This was a relatively straightforward fix that I couldn't have done if I didn't think to look around the installed files and make some changes myself.
