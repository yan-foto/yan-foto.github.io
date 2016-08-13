---
layout:     post
title:      The Mystery of jQuery in Nodejs
date:       2015-07-31 20:42:19
summary:    jQuery is written in Javascript and Node.js is a Javascript runtime, so why using jQuery in Node.js doesn't make sense? (actually it might, but let's all agree that it doesn't)
categories: jquery nodejs node cheerio
---

## Its majesty, the jQuery
There is most probably no web developer who isn't aware of [jQuery](https://jquery.com/): it provides a set of cross-browser wrappers and helper functions which are built upon functions and objects provided by the global context (e.g. `ajax` wrapping `window.XMLHttpRequest`) and in addition it has some neat features for HTML/XML parsing (e.g. `$('<div class="some">fancy stuff</div>')`) and DOM manipulation.

But [what does it really mean](http://stackoverflow.com/q/31393714/2295964) when jQuery is used within a Javascript runtime, where the global context is different from that of browsers and does not provide a DOM object (i.e. [`window.document`](https://developer.mozilla.org/en-US/docs/Web/API/document)). In other words, how can we imagine jQuery with no HTML?

## Node.js vs. Browser
The release of Node.js was for some equal to being able to run the code which they had already written for client-side execution in the browser on the server-side. This lead to a lot of confusion, which in turn lead to the genesis of this article.

The difference between Node.js and a browser in terms of Javascript runtime is indicated (in a subtle manner) in the definition of Node, stating that it is a

> platform built on [Chrome's JavaScript runtime](http://code.google.com/p/v8/)

it simply means that it allows you to execute javascript code. Browsers also have their own JS runtime to execute scripts on the client side **and** in addition provide a mean ["for representing and interacting with objects in HTML, XHTML, and XML documents."](https://en.wikipedia.org/wiki/Document_Object_Model) and that is the Document Object Model (DOM).

Another important difference is the [module system of Node.js](https://nodejs.org/api/modules.html) which allows to dynamically import Javascript code:

```js
<script src="./lib/jquery.min.js">  // In browser
var $ = jQuery = require('jquery')(window); // In Node.js
```

It can be seen that actually the code which is meant to run in Node environment will most probably not run in the Browser and vice versa. There are, however, tools (e.g. [browserify](http://browserify.org/)) which make it possible (*not in all cases*) for Node code to run also in the browser, but that is not the topic of this article.

## jQuery and Node.js
[jQuery](https://jquery.com/)'s claim is to

> make things like HTML document traversal and manipulation, event handling, animation, and Ajax much simpler with an easy-to-use API that works across a multitude of browsers.

To understand the mismatch of Node.js and jQuery, lets start with an example, which was also a [question on stackoverflow](http://stackoverflow.com/q/31654977/2295964):

> `var $ = require('jquery');`
>
> Why is `$.getJSON` undefined?

First of all it should be noted that for CommonJS and all other environments (such as Node) which support the module system, jQuery has to be initiated with a `window` instance which provides the `document` object, as it can be seen from the source code:

```js
(function( global, factory ) {

    if ( typeof module === "object" && typeof module.exports === "object" ) {
        // For CommonJS and CommonJS-like environments where a proper `window`
        // is present, execute the factory and get jQuery.
        // For environments that do not have a `window` with a `document`
        // (such as Node.js), expose a factory as module.exports.
        // This accentuates the need for the creation of a real `window`.
        // e.g. var jQuery = require("jquery")(window);
        // See ticket #14549 for more info.
        module.exports = global.document ?
            factory( global, true ) :
            function( w ) {
                if ( !w.document ) {
                    throw new Error( "jQuery requires a window with a document" );
                }
                return factory( w );
            };
    } else {
        factory( global );
    }

// Pass this if window is not defined yet
}(typeof window !== "undefined" ? window : this, function( window, noGlobal ) {
  // implementation

  return jQuery;
}));
```
for example using [jsdom](https://github.com/tmpvar/jsdom) (version `3.x`), you could fetch a web page (or directly provide the HTML code) and let `jsdom` mock a browser environment and provide access to its global context (i.e. `window`):

```js
var jsdom = require('jsdom');
var $ = null;

jsdom.env(
 "http://quaintous.com",
 function (err, window) {
   $ = require('jquery')(window);
 }
);
```

In other words if you want to use jQuery inside Node you should have a DOM object already prepared and use it to initialize jQuery. Afterwards you can make use of selectors, etc.

# What now?
Now is time to think out of the box, to go through [five stages of grief](https://en.wikipedia.org/wiki/K%C3%BCbler-Ross_model) ASAP and accept that jQuery is not the best for server-side solutions. Asides from requiring to have access to an actual DOM object, it is delivered with loads of other functionalities which are of no use in Node.js environment. There are definitely better solutions!

In one of my latest projects I came across [cheerio](https://github.com/cheeriojs/cheerio) and got to appreciate it very quickly. My goal was to fetch an HTML page, use CSS selectors to select rows of a specific table and to extract required information. As it turned out, the solution was pretty straight forward! The following snippet fetches the main page of this site and lists the (relative) links to the latest articles:

```js
var request = require('request');
var cheerio = require('cheerio');

request('http://quaintous.com', function (error, response, body) {
  if (!error && response.statusCode == 200) {
    var $ = cheerio.load(body);
    $('.post-link').each(function (index, el) {
      console.log(el.attribs.href);
    });
  }
});
```
