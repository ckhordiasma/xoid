---
layout: project
title: "Performance Report Tool"
date: 2019-11-05
tags:
- React
- Bulma
- HTML
- CSS
- Javascript
- GitHub Actions
- DynamoDB
- Elastic Beanstalk
- AWS Lambda
---

The Air Force has specific, documented processes for submitting performance reports and award nominations using standardized PDF forms. Within this process, several “unspoken rules” naturally emerged on how to craft sentences, or “bullet points,” on these forms, which usually involved:

- Making the text visually appealing by leaving minimal space at the end of lines
- Making the text information-dense by including as many abbreviations as possible. 
- Minimizing redundancy in terms of the nouns, verbs, and adjectives that are used

The PDF form had input fields for the bullets that are 3 or 4 lines long, with text overflow disabled.. This made it very cumbersome to do iterative writing in the form, as you would suddenly stop being able to enter text at times. And the unspoken rules of prettiness incentivized people to edit directly in the form instead of somewhere sane like MS Word.

In addition, to achieve maximal visual appeal, people would manually replace normal spaces with unicode whitespace characters of different widths to make the lines fit nicer into the lines. People generated thinner variants of a white space by going into MS Word, typing alt + x over a highlighted "2009", and copying the result.

I wrote a frontend web application using React to that could do these “unspoken rules” for me. For spacing, it would do the following

1. measure the width of a bullet point
2. compare this width to the expected width of the pdf form input field
3. insert special whitespace characters as necessary to shrink/expand the bullet to fit the line better. 

This allowed me to create bullet points that fit perfectly in the pdf form input. The user interface, since it was a mostly accurate replica of the PDF form, had an added benefit of minimizing the amount of time spent dealing with the pdf form. 

After gathering feedback from my office coworkers, I added some more features: 

- A user-defined list of abbreviations and automatically abbreviation of words
- Instant thesaurus lookup on highlighted words
- Save, import, and export capabilities

I originally made the tool for me and my team to use, but later decided to release it to the public internet for anybody to use. I was surprised and happy to see that it became very popular! I started regularly getting emails from Air Force military members around the world who were thanking me for making this tool. I added some metrics tracking to the website, and found that hundreds of thousands of people were using my application every week!

Around 2021, the Air Force completely changed their policies for report writing, changing to a narrative style instead of bullets. The new guidance also included specific instructions that stated that extra space at the end of lines was acceptable. 

This pretty much made my project obsolete, but I was overall very happy with this project. I was able to identify and alleviate a problem shared among a large group of people, and it was great to be able to see all the positive feedback I got from it. 

On the technical side this project taught me how to use React, Github Actions, AWS DynamoDB, and AWS lambda. If I had any regrets, it would be that I wished I had done much more comprehensive unit testing. After I released the application to the public and it became popularized, I was very hesitant to make broad changes that could possibly make things worse; unit tests would have really helped for me to deploy with confidence during those times.