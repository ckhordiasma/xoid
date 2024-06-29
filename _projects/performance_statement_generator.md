---
layout: project
title: "Performance Statement Generator"
date: 2024-05-27
tags:
- MLOps
- Machine Learning
- Tensorflow
- RNN
- Web Scraping
---

![image of ui](/media/bullet-synth/ui.png)

project page: <https://github.com/AF-VCD/bullet-synth>

demo page: <https://af-vcd.github.io/bullet-synth>

The Air Force has specific, documented processes for submitting performance reports, and after many years of people following these processes, a unique "style" of writing performance bullet statements emerged. I was discussing this phenomenon one day with my teammates, and somebody brought up that maybe if we supplied enough examples, we could train a machine learning model to capture the essence of Air Force bullet statement writing. That was the genesis of this project.

This was several years before the emergence of ChatGPT, which would have been an easier answer to this problem. The cutting edge at the time was Long-Short Term Memory / Recurrent Neural Networks, and I found a tensorflow example from the tensorflow website [here](https://www.tensorflow.org/text/tutorials/text_generation) that I could adapt for this project. 

### Data 

Fortunately, there were already tons of bullet statements online that I could use for training. I wrote some python scripts to scrape all this data for me, and a perl script to clean up the data a bit. The github project for that is [here](https://github.com/AF-VCD/bullet-scraper). 

### Training

The machine learning models were built using Tensorflow in Python/Jupyter notebooks. Through a process of trial and error, I ended up creating four models. The models varied in the number of LSTM layers, LSTM neurons, and training epoch sizes, but all used the same training dataset. GTX 2070 and GTX 970 graphics cards were used to train the models. The models took around two and a half hours to train each.

### Deployment

In order to get predictions working on the web, I used a python script to convert the Tensorflow models into ones readable by Tensorflow.js, uploaded the models to github, and deployed using GitHub pages.

### Quality

My expectations for this project were low. I was going to be happy if the models produced any output, even if it was nonsensical. It turned out to be mostly nonsensical, but because normal Air Force bullet statements are also mostly nonsensical, it ended up being pretty good at capturing the style of bullets. 