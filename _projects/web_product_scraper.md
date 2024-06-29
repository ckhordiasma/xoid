---
layout: project
title: "Web Product Scraper"
date: 2024-05-27
---

 Retail Arbitrage is a way you can make money by being a middle-man. You can look for and buy cheap products at brick-and-mortar stores in bulk and then sell them on other marketplaces like Amazon, where there is always demand/sales volume. My wife does retail arbitrage, and wanted to expand into online arbitrage, which is the same thing but buying from online stores instead. She came to me for some help to be able to do this more efficiently. 

Her typical workflow for online retail arbitrage was as follows

1. look through an online store (i.e. home depot) and see what items they have on discount. For a given item:
2. Verify the item is sold on Amazon
3. Verify the item has sufficient sales volume on Amazon
4. Verify the item's profit margin + sales volume combination makes it a good candidate for purchase
u
She wanted to automate this entire process, and she wanted me to do it. After some negotiation (I wanted to buy the new Framework 16 laptop for myself), I started work on this project.

## Design

There are two main components in my solution architecture -  code that scrapes websites and code that talks to the Amazon Seller API. 

### Database

I used a redis database to pass information between these two halves. Why redis, you ask? a couple of reasons:

- I saw the fireship.io short video on redis and wanted to try it out
- i don't really care about the data in the database long-term, but do care that the database is fast

### Web Scraper

Web scraping can be challenging and very ad-hoc at times. Each website is going to require different logic and code to get the desired information. I designed the scraper component to be very modular, so that programming the scraping logic for one particular site could be decoupled from other sites as well as the rest of the application. 

### Scraping methods

Sites vary on how amenable they are to scraping. Some will accept all GET requests, as long as the requests from a particular ip address isn't above a certain rate threshold. Adding some rate limiting to the scraper will prevent it from getting flagged by this.

Other sites will block certain requests based on volume from a given ip address and user-agent string. This can be circumvented by randomizing the user-agent string on every request, making it look like several different devices are accessing the site from behind a NAT gateway.

Some sites, though, have more sophisticated anti-bot/scraper tools, which make your browser do some javascript and maybe some API calls before producing the actual content of the page. For these sites, I have needed to resort to using a third-party scraping tool called Zenrows to get page content for me. 

Once I have a page's HTML content, I judiciously use DOM traversing techniques or regex to get the specific info I need. The key piece of info needed for each scraped product is a UPC code, which can be used to cross-reference the product with an Amazon listing.

Besides parsing HTML, another way to get good scrapes is to examine your browser's network activity when visiting a given site. Sometimes I have found that my browser is actually making calls directly to an API server to get product information, and I can use dev tools to get the GET/POST parameters, as well as any API keys that were used! These exposed APIs are great because they usually don't have advanced anti-bot protections on them.

### Amazon Lookup

Amazon has an entire suite of APIs, known as the Amazon Selling Partner (SP) API, for getting things like product listing information and pricing. They have pretty comprehensive documentation on it too: <https://developer-docs.amazon.com/sp-api/docs/welcome> 

Getting started with this API was a little tricky because it wasn't as simple as adding an `Authorization: Bearer` header to my requests. You have to set up an IAM account, assume the role of your Selling Partner application, and need to sign every request that you make. Having the AWS SDK helps make this go smoother though. 

Using the API, I was able to search by UPC to get a list of amazon standard identification numbers (ASINs), and then query those ASINs to get more details about the products, including their sale prices, sales rankings, and estimated shipping costs. 

All of this information can be factored in to an equation to estimate how profitable a given item is. Once I collect all this product data and produce a profitability score for each one, I can give my spouse a summary spreadsheet, and she can do a quick double check of the info before deciding what to buy for retail arbitrage. 

