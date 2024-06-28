---
layout: project
title: "Web Product Scraper"
date: 2024-05-27
---

My spouse does retail arbitrage on Amazon. Basically this means that they look for and buy cheap products at brick-and-mortar stores in bulk and then sell them on Amazon, where there is always demand/sales volume, for full price. They wanted to expand into online arbitrage, which is the same thing but buying from online stores instead; and this is when she enlisted my help. 

Her typical workflow for online retail arbitrage was as follows

1. look through an online store (i.e. home depot) and see what items they have on discount. For a given item:
2. Verify the item is sold on Amazon
3. Verify the item has sufficient sales volume on Amazon
4. Verify the item's profit margin + sales volume combination makes it a good candidate for purchase

She wanted to automate this entire process, and she wanted me to do it. After some negotiation (I wanted to buy the new Framework 16 laptop for myself), I started work on this project.

### Design

There are two main components in my solution architecture -  code that scrapes websites and code that talks to the Amazon Seller API. I used a redis database to pass information between these two halves. Why redis, you ask? a couple of reasons:

- I saw the fireship.io short video on redis and wanted to try it out
- i don't really care about the data in the database long-term, but do care that the database is fast



