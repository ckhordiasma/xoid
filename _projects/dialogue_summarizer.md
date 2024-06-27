---
layout: project
title: "Dialogue Summarizer with Tuned FLAN-T5 Model"
tags: 
- Amazon SageMaker
- Large Language Model
- Parameter Efficient Fine Tuning
- Low Rank Adaptation
- Soft Prompting
date: 2024-06-06
---

Recently, I had the opportunity to take the ["Generative AI with Large Language Models (LLMs)"](https://www.coursera.org/learn/generative-ai-with-llms) course. As part of the course, I got the opportunity to create several fine-tuned LLMs using Amazon SageMaker. Overall I thought this course was great for me, a person with a basic machine learning background (took Andrew Ng's machine learning course on coursera back in 2012, and have done a few personal projects with neural networks) and have experience coding in Python. The course did a good job of explaining the techical concepts behind LLMs without getting too bogged down in the weeds. The hands on activities were great, with a good integration with lab accounts in AWS to actually run notebooks and things. The purpose of this post is to document a few key things that I learned regarding building fine-tuned LLMs based on foundational models.

## Parameter-Efficient Fine Tuning (PEFT)

LLMs can be huge, and are computationally expensive to train. This is why most people use a foudnational model as a starting point. However, they can also be expensive to fine-tune, if you do it the traditional way of updating all model weights based on new training data. Parameter Efficient Fine Tuning (PEFT) are a class of techniques you can use to fine tune a model without trying to tune all the billions of parameters of a standard LLM. The course covered two PEFT methods: Soft Prompting and Low Rank Adaptation. 

### Soft Prompting

In LLMs using a transformer architecture, the computational path from prompt text to encoder self-attention looks something like this:

1. Text is converted into numbered tokens
2. Tokens are translated to vectors based on a vector space representation of the language (embedding)
3. Tokens are translated to vectors based on a vector space representation of the position of the tokens in the prompt (position)
4. Embedding and position vectors are summed for each token to create the final embedding vectors
5. Vectors are multiplied by a weighted matrix to calculate the self-attention matrix for the words

Soft prompting injects a step around the embedding step by adding embedding vectors before the tokenized/vectorized prompt input. These vectors sit in the language vector space but don't necessarily correspond to a specific word (might have words of a similar meaning clustered around it though). It's kind of like adding "magic words" to the beginning of your prompt that magically make your model better. Except it's not magic, it's math, and you have to train the model to learn what those magic words are supposed to be. Typically about 20-100 tokens/magic words are used to achieve good results. If you had a vector embedding space of size 512, then worst case that would be 51,200 parameters to train. That's a lot more manageable than the billions of parameters that may have been in the original model!

According to a paper that looked at this (Lester et al. 2021, "The Power of Scale for Parameter-Efficient Prompt Tuning"), prompt tuning works less well for the millions-of-parameters model, but works super well for billions and tens-of-billions parameter models (when compared to full fine-tuning)

### Low Rank Adaptation (LoRA)

LoRA seems to be the most popular PEFT choice, and apparently when people say PEFT they usually specifically mean LoRA. In LoRA, the fine-tuning happens during the computation of self attention from the embedded input vectors. Instead of re-training the weighted matrix before self attention, in LoRA you train a couple of smaller matrices that, when you multiply them together, creates a matrix that is the same size as the original weighted matrix. Then you add this new matrix with the original weighted matrix and use this instead to calculate self-attention. This means that every parameter of the original matrix is shifted by the dot product of a unique row/column vector combination from the two smaller matrices. 

People (Hu et al. 2121, "LoRA: Low-Rank Adaptation of Large Language Models) have found that using a rank--the smaller dimension of the two LoRA matrices--of 4 or 8 can achieve good results, with anything above that not being much better. 

## Leveraging PEFT 

PEFT is exciting because it means someone like me, who doesn't have millions of dollars lying around, has a chance at making useful LLMs from large foundational models at a fraction of the cost. In this course we were provided Amazon SageMaker Studio (jupyter environment) with ml.m5.2xlarge instances and two hours to complete each lab. According to current AWS pricing, these instances cost $0.461/hour. The fine-tuning was done on the flan-T5 base model, which has 248M params. Even with a model this small, the compute instance was not able to do fine-tuning or LoRA in a reasonable timeframe. Just a single full fine-tuning optimization step on 1% of the training data took several minutes. I was able to do 5 epochs/100 steps of LoRA optimization on the 1% of training data, but the results were not that great

Here are some results from the lab, which skipped most of the actual training and just downloaded models that were trained beforehand:

```
---------------------------------------------------------------------------------------------------
BASELINE HUMAN SUMMARY:
#Person1# teaches #Person2# how to upgrade software and hardware in #Person2#'s system.
---------------------------------------------------------------------------------------------------
ORIGINAL MODEL:
@Person1## would like to upgrade his system to a computer.
---------------------------------------------------------------------------------------------------
INSTRUCT MODEL:
#Person1# suggests #Person2# upgrading #Person2#'s system, hardware, and CD-ROM drive. #Person2# thinks it's great.
---------------------------------------------------------------------------------------------------
PEFT MODEL: #Person1# recommends adding a painting program to #Person2#'s software and upgrading hardware. #Person2# also wants to upgrade the hardware because it's outdated now.
```

Here is my result on the same prompt when I ran PEFT/LoRA fine tuning with 5 epochs/100 runs max on only 1% of the training data (it took about an hour and 30 minutes).

```
---------------------------------------------------------------------------------------------------
PEFT MODEL: You might want to upgrade your hardware because it is pretty outdated now. #Person1# would probably need a faster processor, to begin with. #Person2# might need a faster hard disc, more memory and a faster modem. #Person2# might want to add a CD-ROM drive too, because most new software programs are coming out on CDs.
```

## Running on Google Cloud

To get around having the two hour lab limit, and because I had some promo Google Cloud credits, I decided to try running this notebook on Google Cloud Colab Enterprise instead. To emulate the specs of the Amazon ml.m5.2xlarge, I ended up choosing a n2-standard-8 ($0.388472/hr) with an 100GB SSD drive ($17/month)


With a new environment, I tried redoing the LoRA fine tuning over the entire training dataset instead of just 1%. I initially tried the following training settings (based on a comment from the instructor in the course) 

```
  learning_rate=1e-3, 
  num_train_epochs=5,
  max_steps=100
```

but was not getting good results.. so I tried adjusting these (hyper?)parameters a bit. I quickly learned that iteratively adjusting these parameters was going to take too long with just a CPU (it was projecting over 100 hours in some cases!), so I switched my environment to utilize a GPU instead. Specifically I chose the g2-standard-8 VM, which comes with a NVIDIA_L4 GPU, and a 200GB SSD drive for good measure. 

With this my training was happening MUCH faster, like two training steps a second instead of a training step every thirty seconds before. I used the following parameters:

```
  learning_rate=1e-4,
  num_train_epochs=5, # note: max_steps unset, which means it's set to unlimited
  logging_steps=100 # to reduce output spam a bit
```

and trained for about 1:24 hours / 7790 training steps, and I got quite good results!

```
---------------------------------------------------------------------------------------------------
BASELINE HUMAN SUMMARY:
#Person1# teaches #Person2# how to upgrade software and hardware in #Person2#'s system.
---------------------------------------------------------------------------------------------------
PEFT MODEL (trained by me): #Person2# wants to upgrade #Person2#'s system and hardware. #Person1# recommends adding a painting program to #Person2#'s software and adding a CD-ROM drive.
```

In all, including my trial and error using different compute VMs, I spent $15.65 training this model. If I were to drill down on just the costs that were associated with the GPU training, It was $5.60 for the GPU, $2 for the CPU, $0.94 for the RAM, and $0.44 for SSD, for a grand total of $8.98. 

However, I also accidentally left my compute instance on overnight, so just to make sure, I went back and reran the notebook, terminating the instance once it was done. Then I checked what the differential cost was: 