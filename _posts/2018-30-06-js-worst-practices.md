---
layout:     post
title:      JavaScript Bad Practices
date:       2018-06-30 6:30:19
summary:    If you're new to JS you might be used to some practices which are not idiomatic to JS. This article summarizes two common mistakes that I encountered during refactoring.
categories: js nodejs practice
---

## Introduction
Recently I was assigned to refactor a JS backend project. The original developer/maintainer has a background in developing frontend with AngularJS. Although the server was working as expected, it contained a number of recurring bad practices, which I am going to summarize here in form of *Do*s and *Don't*s.

## Using `global`
Those (including myself) who learnt JS in the context of web browsers, are familiar with the `window` object which implicitly contains all global variables, functions, etc. In NodeJs the [`global`](https://nodejs.org/api/globals.html#globals_global) object is the global scope and defining variables at the top-level scope does not automatically attach them to the `global` object. Using global *variables* ([and not necessarily *constants*](https://stackoverflow.com/a/1265684/2295964)) is considered bad practice (see [this very informative article](http://wiki.c2.com/?GlobalVariablesAreBad)).

### *Don't*
```js
// index.js
let Log = require('./my-logger')
global.log = new Log('my-app')

// other.js
log.info('I am using a global object!')
```

The example above shows how initializing a logger in `index.js` and using it in `other.js` makes use of the `global` object.

### *Do*
There is no one-size-fit-all approach on how to avoid `global` but depending on your usage it can vary between passing variables as function parameters to exporting them through modules.

## Misusing `require`
During my refactoring session I came across some *orphan* `require`s, that is `require`s within the code (and not at the top of the module) without any returns.

### *Don't*

```js
function otherthing() {
  // do something
  require('./something1')
  require('./something2')
  // do the rest
}
```

It turned out that the developer was trying to modularize the code in the tradition of *ancient* programming languages where directives such as `include` would copy the content of another file at the position where the directive is called. To make it more confusing, the required files made use of `global` to access objects which were declared in a separate file.

### *Do*

```js
let something1 = require('./something1')
let something2 = require('./something2')

function otherthing(param1, param2) {
  // do something
  let result1 = something1(param1)
  let result2 = something2(param2)
  // do the rest and use results
}

module.exports = otherthing
```

## Useless promises
If you are familar to concept of concurrency in terms of threads (e.g. Java, C++), processes (e.g. Erlang), etc. you might have some difficulties understanding it in terms of [single-threaded event-loop based NodeJS](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/): "The event loop is what allows Node.js to perform non-blocking I/O operations".

In JavaScript concurrency is take care for you: blocking I/O operations are handled using callbacks, promises or by awaiting on async function. This, however, does not mean that wrapping your code inside a promise automagically execute it concurrently:

```js
console.log('before')
let p1 = new Promise((resolve, reject) => {
  // Some heavy computing
  console.log('inside')
})
console.log('after')
```

The code above would produce `before`, `inside`, and `after` (in that order) even with the promise `p1` not bein called.

## Chaining promises
[Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) are an effective remedy to [callback hell](http://callbackhell.com/). So if you want to chain promises:

### *Don't*
```js
p1()
  .then(p1Result => {
    // p1 handling
    p2().then(p2Result => {
      // p2 handling
      // and so on...
    })
    .catch(err => {
      // p2 errors
    })
  })
  .catch(err => {
    // p1 errors
  })
```

### *Do*
```js
p1()
  .then(p1Result => {
    // p1 handling
    return p2()
  })
  .then(p2Result => {
    // p2 handling
    // and so on...
  })
  .catch(err => {
    // p1 and p2 errors
  })
```

A cleaner solution would be to use [`async/await`](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx):

```js
async function run() {
  try {
    let p1Result = await p1()
    let p2Result = await p2()
    // and so on...
  } catch (err) {
    // p1 and p2 errors
  }
}

run()
```
## Array operations
Not only in JS, but also in other programming languages following functional paradigms, provide a number of methods to iterate and transform arrays. One of the most curios patterns that I encountered during the following was the following:

### *Don't*
```js
let a = [1,2,3,4]
for (key in a) {
  let index = parseInt(key)
  // Do something
  if (index++ === a.length) {
    // Last iteration here
  }
}
```

It should be noted that [`for...in` ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in) is meant to iterate over `Object` keys and not in combination with arrays. `parseInt` was used here as `key` is always an string in `for...in` loops.

### *Do*
Depending on what you want to, you can avoid `for`s altogether by using following [`Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array) methods:

 * [`map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) to transform an array:
   ```js
   // Calculating root square of all elements in an array
   let a = [1,4,9]
   a.map(el => Math.sqrt(el)) // [1,2,3]
   ```
 * [`every`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/every) to verify if all elements of an array satsify a given condition:
   ```js
   // Check if members are all even
   let a = [1,2,3,4]
   a.every(el => el % 2 === 0) // false
   ```
 * [`find`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find) to find the first element satisfiying a given condition:
   ```js
   let a = [1, '2', {3: true}]
   a.find(el => typeof el === 'string')    // '2'
   a.find(el => typeof el === 'function')  // undefined
   ```
 * [`reduce`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) to *summarize* an array into a single variable:
   ```js
   // Create an object from a two-dimensional key-value array
   let a = [['name', 'reduce'], ['type', 'function'], ['value', 42]]
   a.reduce((acc, cur) => {
     acc[cur[0]] = cur[1]
     return acc
   }, {}) // { name: 'reduce', type: 'function', value: 42 }
   ```
  * [`some`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some) to check if at least an element in array fulfills the condition:
   ```js
   // Check if at least on of the funcions retuns hello world
   let a = [() => 42, () => 'hello world', () => 'goodbye world']
   a.some(el => el() === 'hello world')
   ```

Just remember: using aforementioned methods adds an overhead to your program which impacts performance comparing to basic usage of `for` loops.

## Summary
In this article concurrency and functional programming in JavaScript were addressed briefly. The most important lesson is 1) concurrency is taken care for you, 2) functional programming simplifies working with arrays.
