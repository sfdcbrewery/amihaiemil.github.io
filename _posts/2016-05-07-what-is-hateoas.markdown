---
layout: post
title:  "What is HATEOAS?"
date:   2016-05-07
categories: rest
---

This article assumes you have at least an intermediate knowledge about REST. If you don't, there are [many](https://www.google.ro/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=rest%20tutorial) tutorials that teach you how to implement a RESTful API. I learned REST from Bill Burke's [RESTful Java with JAX-RS 2.0](http://shop.oreilly.com/product/0636920028925.do), which I would recommend to any Java EE developer starting to learn this architecture. Also, Github's [API](https://api.github.com/) serves as agreat example.

In his book Burke also describes HATEOAS (hypermedia as the engine of application state). It is what's known as the 3rd level of REST [maturity](http://martinfowler.com/articles/richardsonMaturityModel.html). Basically, the idea is that you can implement web services as simple as possible by using a single HTTP method and nothing else, or you can make use of all the power that HTTP gives you (different methods, status codes, mime-types etc) to make your services truly RESTful. 

Now, everyone tells you about mime-types, about how your API should be **navigable** but I've seen few less-talk-more-concrete examples. So that's what this article is really about: **a practical example**.

Let's assume you have to design an endpoint for search which takes a **q** parameter, representing the query (keywords). All good, you write your code and your service is accessible at **GET http://example.com/rest/search?q=(...)**. This will return an array of JSON results of format: 
{% highlight json %}
[
...,
{
    "title": "Page title",
    "link": "/link/to/page",
    "preview": "text preview",
    "lastmodified": "15/03/2016"
},
...
]
{% endhighlight %}
It does use HTTP, it returns a JSON structure, it's exposed for the world to consume, so we can call it a REST service, right? Yes, but it's far from its true potential. 

**First problem**: no pagination. So for now, clients will call this endpoint and probably get a huge amount of results which they'll have to handle and paginate by themselves. Let's introduce parameters **index** and **nr**. This way, clients' work is already leveraged a lot: they'll call **/rest/search?q=test&index=0&nr=10** and get the first 10 results. To get the following 10, they'll add 10 to **index**. Nice, but there is still a misunderstanding: how many results are there? How long does **index** go? We also have to return the total number of results found, say **resultsFound**.
Your JSON response now looks like this:

{% highlight json %}
{
    "resultsFound": 123,
    "results": [
                 ...,
                 {
                     "title": "Page title",
                     "link": "/link/to/page",
                     "preview": "text preview",
                     "lastmodified": "15/03/2016"
                  },
                  ...
                ]
}
{% endhighlight %}

Now the client is not flooded with data and can do the maths to figure out the number of pages. HATEOAS enough? No - it's not yet **navigable** . 
We can leverage the client's work even more if we add **previousPage** and **nextPage**. We have the **index** and **nr** from the client's call, we know the number of results, so we can build the links to the immediate pages. If we are on the first page, then **previousPage** will be empty (**-**) and if we are on the last page, **nextPage** will be empty. 

Having made these changes also, your 3rd page of results (with 10 results/page) looks like this: 

{% highlight json %}
{
    "resultsFound": 123,
    "previousPage": "/rest/search?q=test&index=10&nr=10",
    "nextPage": "/rest/search?q=test&index=30&nr=10",
    "results": [
                 ...,
                 {
                     "title": "Page title",
                     "link": "/link/to/page",
                     "preview": "text preview",
                     "lastmodified": "15/03/2016"
                  },
                  ...
                ]
}
{% endhighlight %}

The response can now be navigated. However, I want to see the last page of results. Do I click 12 times on **nextPage**? Besides, almost all the sites on the web show links to at least the first 5-10 results pages pages. Let's add the array **pages** containing all the links of this search (we can assume the client will want to show at least the first 10 pages, so we spare him a worry).

{% highlight json %}
{
    "resultsFound": 123,
    "previousPage": "/rest/search?q=test&index=10&nr=10",
    "nextPage": "/rest/search?q=test&index=30&nr=10",
    "results": [
                 ...,
                 {
                     "title": "Page title",
                     "link": "/link/to/page",
                     "preview": "text preview",
                     "lastmodified": "15/03/2016"
                  },
                  ...
                ],
    "pages": [
                 "/rest/search?q=test&index=0&nr=10",
                 "/rest/search?q=test&index=10&nr=10",
                 ...,
                 "/rest/search?q=test&index=120&nr=10"
             ]
}
{% endhighlight %}

Your JSON response is now both navigable and scalabale! The client can easily choose how many results are there on a page and has an instant and lightweight overview of the whole search.

**Second problem**: status codes. This is also important. Don't use only 200 ok or 404 NOT FOUND! Offer more meaning to your responses, help the clients "understand" everything better. Besides, it's all about **respecting the HTTP protocol**. If there are no results found, return an empty JSON, but set the status appropriately, to **204 NO CONTENT**. If there are validation errors (say **index** is out of range), return **412 PRECONDITION FAILED** or **422 UNPROCESSABLE ENTITY**. And the list can go on. You got the idea: respect the protocol, make the communication as easy as possible. An alternative to a propper status response would be to return a message, for instance:

{% highlight json %}
HTTP Status 200 OK
{
    "message": "no results"
}
{% endhighlight %}

But this is neither HATEOAS nor true HTTP. A correct response would be:

{% highlight json %}
HTTP Status 204 NO CONTENT

{
    "resultsFound": 0,
    "previousPage": "-",
    "nextPage": "-",
    "results": [],
    "pages": []
}
{% endhighlight %}

I hope you now have a better understanding of the HATEOAS principle. Your search endpoint is now much more usable in any context. Wheter it will be consumed by a Javascript client or by a Java/C#/Python/Whatever library, the communication will be smooth, and the client's code fluent and less bug-prompt. 

You can also think of it (slightly) as an HTML page that's been stripped of its CSS. Look at the search response above. Doesn't it look like an HTML page that lost its styling? Now, think of how easy it will be for any JS client to work with it: simply make a call to the endpoint, get the JSON object and display the text and links. 
  
