---
layout: project
title: "Resume Build Pipeline"
date: 2024-05-27
tags:
- Github Actions
- LaTeX
- Bash
- CI/CD
---
[link to github project](https://github.com/ckhordiasma/resume-build-example)

I decided to separate from the Air Force in 2024, which meant that I had to start preparing my resume and start applying for jobs. I thought it would be cool if I could version-control my resume, So I started writing my resume in LaTeX and committing the .tex files to a git repo. 

This worked out great! I kept a log in Notion of all my job applications, and I could add entries with the resume commit hash that I used for any given job application. If I needed to prepare for an interview and wanted to see what version of my resume I submitted for that company, I could checkout the referenced commit and build the pdf from there.

... but after doing this for several job applications and resume iterations, it started to get cumbersome. I was doing a lot of manual building of PDFs from .tex source files, and it was getting kind of hard to keep track of pdfs on my local machine. Also, just seeing a commit hash in my log entries gave me little information about what my resume was like at that point.

This led to me implementing a github actions pipeline for my LaTeX repo. In the actions pipeline, a LaTeX container would take my code and build it into one or more pdfs. A second action would take the pdf artifact(s) and push them as github releases in my repo. Initially for convenience I had the pipeline running on every pull request, and releases tagged by the github actions job number, but later I switched the actions pipeline to run whenever "vN.NNN" tags are created, so I have more control over what becomes an actual release. In my notes I can also just put the tag version number instead of the commit hash, which is a lot shorter and more understandable. 

My resume repository is private, but by this point I figured I should showcase what I did here, so I spun off the code into an example respository and wrote up this post too.