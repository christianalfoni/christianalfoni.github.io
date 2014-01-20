---
layout: post
title:  "Client <-> JSON <-> NodeJS"
date:   2014-01-20 09:21:30
categories: javascript
tags: ["javascript", "json", "node"]
---

### What is HTTP?

To get an understanding of why JSON is a good data-structure for communcation between the client and the server you have to know about
the HTTP protocol.

HTTP (HyperText Transfer Protocol) started out as a protocol to transfer text (string). You would hit an address (url) and the client got
a string back. The string could contain information about other addresses (urls) and there you have it... a webpage.

When it comes down to it, HTTP is just a way of transferring data between two applications, normally a browser and a server application, but
it can also be used between two different server applications. It is a request/response protocol, meaning that you ask for something and you
get something back and the type of data transferred is just a string. HTTP has no understanding of integers, booleans etc.

### How do you use HTTP?

A HTTP **request** has the following content:

- **Method** (GET, POST, PUT, DELETE etc.)
- **URL**
    - http://localhost:9000/contacts
- **Query**
    - http://localhost:9000/contacts**?type=private&limit=20**
    - The query begins with a question character (?)
    - Are key/value pairs seperated by **&**, e.q.: limit=20&filter=abc
- **Headers**
    - User-agent
    - Content-type
    - X-requested-with
    - Cache-control
    - Cookie
        - Can be multiple cookies
        - Also key/value pairs, only seperated by **;**
    - And lots more...
- **Body**

A HTTP **response** has the following content:

- **Status code**
- **Headers**
    - Access-Control-Allow-Origin
    - Content-length
    - Content-Encoding
    - Content-type
    - Cache-control
    - Set-Cookie
        - E.g.: UderId=CrazyPants;Max-Age=3600;
        - Newline is the delimiter
        - Whatever cookie is set will be included on all following requests
    - And lots more...
- **Body**

### So what does the server do when it gets an HTTP request?

A request is sent from a browser to the server. As a developer you depend on a library that is built to handle the request.
Many libraries does a lot of the work for you and basically the following is happening:

- Parses the URL to match any defined routes
- Parses any cookies passed in on the request
- Parses the query of the request
- Parses the content of the body, based on the **content-type** passed in
- And lots more

When the request is handled it will be responded with the following:

- A status code
- Headers defined by the library you are using and any headers you have set yourself
- Among the headers, the content-type which tells the client how to parse the content of the body
- The body, which is just a string

### Sending complex data to the server

The data-structures of our requests and responses has become more complex, as more complex applications are being built on the web.
When the web first started out it was all about **forms**. Inputs, dropdowns, checkboxes, radiobuttons and textareas. Basically they
could handle the following data:

- strings
- numbers
- booleans
- lists (comma-separated)

But as the web has evolved, the need to send more complex data has too. Looking at JavaScript:

{% highlight javascript %}

    // Typical form data
    data: {
      myString: 'string',
      myNumber: 20,
      myBoolean: true,
      myList: ['string', 'string']
    }

    // Typical complex data
    data: {
        myObject: {
            myString: 'string',
            myNumber: 20,
            myBoolean: true,
        },
        myObjectList: [{
            myString: 'string',
            myNumber: 20,
            myBoolean: true,
        }]
    }

{% endhighlight %}

To sum this up quickly, the need to pass objects between the client and server has made things more complex, but oh so much more powerful.

### Our options for passing data

By defaults any data sent to the server is **application/x-www-form-urlencoded**. If you have Node JS running
