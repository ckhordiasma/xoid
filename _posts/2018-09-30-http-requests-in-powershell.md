---
layout: post
title: "HTTP GET and POST Requests using PowerShell"
date: 2018-09-30
post_id: 
---
I have been doing some work using PowerShell to automatically scrape website content. 

If you have a direct link to some file (let's say, a word document uploaded online), then one can do

```powershell
Invoke-WebRequest -Uri "http://aaa.com/smthg.docx" -Outfile "./download.docx" # -Method GET is implied
```

This will quickly and easily download the file located at `http://aaa.com/smthg.docx` 
and place it in the current working directory as `download.docx` .

However, a lot of times files won't be very nicely available as direct links, but rather are returned only after some user input/interaction,
like a form submission. In that case, you have to be a little fancier. To learn how to do this, I used a sample website called [quotes.toscrape.com](http://quotes.toscrape.com/search.aspx).

The specific problem I was trying to solve was to programmatically (i.e. NOT with a web browser and mouse clicks) navigate ASP.NET-based websites, which typically
have the file extension .aspx . 
What I discovered was that ASPX sites usually use hidden form variables to keep track of where the user is in a website. One such variable 
is called `__VIEWSTATE` (yes, that's two underscores), and it's usually some really long string that represents some data that has been encoded in base-64. The Quotes to Scrape website is an 
ASPX website that utilizes `__VIEWSTATE`, so it was a perfect test site for me to learn how to do this.

Google Chrome and other modern browsers have an "inspect" feature that is very useful in understanding all of the behind-the-scenes stuff that happens
when a website loads. For this example, I primarily examined the "Network" tab, looking at the information on the "search.aspx" request. The following image 
shows all of the header informmation that I see when I first access the quotes website.

(image here)

A lot of the fields there are out of the scope of what I cared about. The main thing I looked at was the first two things: `Request URL` and `Request Method`. These two things
basically say that my browser said `GET` to the server `http://quotes.toscrape.com/search.aspx`, and the server replied by giving back all of the HTML 
that comprised the website. 

In PowerShell, we can do pretty much the same thing with the following commands.
```powershell
$searchSite = 'http://quotes.toscrape.com/search.aspx'
$request = Invoke-WebRequest -Uri "$searchSite" -Method GET
```

All the stuff that the server gives me back will be stored in the `$request` variable. The HTML content of the site can be viewed with `$request.RawContent`.
A lot of the site is split up in pretty logical, understandable parts: for example, form fields can be accessed by running `$request.Forms.Fields`. If you do that command, you should see
something like:

```
Key           Value                                                                                                         
---           -----                                                                                                         
submit_button Search                                                                                                        
__VIEWSTATE   ZDUwNDY1NWVlMjViNGY2NWE2NjUxMDhkZWU2MzdkZDYsQWxiZXJ0IEVpbnN0ZWluLEouSy4gUm93bGluZyxKYW5lIEF1c3RlbixNYXJpbHl...
```

There's that `__VIEWSTATE` variable I was talking about, and if you were to inspect the HTML of the website in the browser, sure enough, right after the submit button is 
defined, there is a hidden input called `__VIEWSTATE` with a bunch of base-64-encoded gibberish as its value.

This is important for what happens next, when an author is selected from the first dropdown menu in the browser. As soon as you do that, the website submits 
a `POST` request to `http://quotes.toscrape.com/filter.aspx`, passing in three input values: `author`, `tag`, and the hidden `__VIEWSTATE` value. This can be verified 
by monitoring the network activity for the `filter.aspx` request in the inspector browser.

(image here)

So how do you do `POST` requests using `Invoke-WebRequest`? Well, first you have to define your POST values in a structure:

```powershell
$postParams = @{
author='Albert Einstein'; 
tag=$request.ParsedHtml.getElementByID('tag').children[0].innerHTML; 
__VIEWSTATE=$request.Forms.Fields.__VIEWSTATE;
};
```

