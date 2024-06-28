---
layout: project
title: "Teachable Text Machine"
date: 2021-04-01
---

[image of UI]

This was a project that I built as part of a research class that I took while I was in the Air Force. I made an application that can create custom text sentiment analysis models using transfer learning on an existing ML model, in this case the universal sentence encoder. By choosing to use transfer learning (like in the original Teachable Machine project by Google), I could run both training and inference in the browser, allowing the entire application to be served as static frontend content. I used Tensorflow.js to train and evaluate the models, and svelte to produce the UI. 

### Design 

I wanted to make it easy for a non-technical person with a basic understanding of machine learning to train their own models with this app. The user can enter in their training samples/categories (i.e. positive, neutral, negative) in the left section, adding sentences delimited by line breaks. The middle section could be used to change the size of each model layer, as well as set some training hyperparameters. After training, the right section could be used to run inference on the model. With the way everything is laid out, it is very easy for a user to modify training categories/samples, tweak model parameters, and evaluate the quality of their model. This level of interactivity allows for faster iteration and overall better ML model development. 

### Model accuracy

Overall, especially with the default parameters and a small model, the model was pretty bad. Here are some example bad results:

[image 1]

[image 2]

I found that if I simplified the problem to just positive/negative, and upped the number of layers and trainig cycles, I was able to get slightly better results. 

[image 3]

 Doing all of this investigation (one might even call it "error analysis" and "design iteration") was easy due to how I built the application, and I could see the results of my changes with just a few button clicks. Even though model wasn't initially that great, I was able to quickly understand how to tweak my data and model parameters to make it better. In this sense I think I was able to achieve my initial design goals for this project.