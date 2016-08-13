---
layout:     post
title:      V8 global proxy object in C++
date:       2015-06-23 19:01:19
summary:    As a JavaScript engine, V8 implements a number of standard JS functionalities, which are available through the global proxy object. These can also be used inside C++ modules that are using V8 interface, such as Node.js addons.
categories: v8 nodejs c++
---

## Motivation
I was writing a [Node.Js addon](https://nodejs.org/api/addons.html) and at some point I wanted to return a JavaScript object. The standard approach would have been to create a [`v8::Object`](http://izs.me/v8-docs/classv8_1_1Object.html) and define each property using [`Set (Handle< Value > key, Handle< Value > value, PropertyAttribute attribs=None)`](http://izs.me/v8-docs/classv8_1_1Object.html#a97717c7b7fdc556c3a7fad14877ca912) method. However, since I already had my stringified object from a REST-API call, the best case would be just to parse the string to an object and return it.

V8 doesn't directly expose a `C++` utility interface to parse JSON string, but we know that most modern browsers provide the [`JSON`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON) object in global context (i.e. `window.JSON`), which also parses JSON string into objects. The idea is to access the global context of V8 and get a hold of `JSON` object and finally call some methods on it.

## Approach
The idea is simple: we get our hands on V8 global context. A [context is](https://developers.google.com/v8/embed#contexts) is

> an execution environment that allows separate, unrelated, JavaScript applications to run in a single instance of V8. You must explicitly specify the context in which you want any JavaScript code to be run.

Getting the global context is as easy as

```cpp
Handle<Context> context = Context::GetCurrent();
Handle<Object> global = context->Global();
```

It can be seen (from `Handle<Object>`), that the variable `global` represents a JS object. This global object resembles `window` object in browser and contains a number of useful utilities. For our use case here, we need to get the `JSON` object first. This can easily be done using the `v8::Object::Get` method:

```cpp
Handle<Object> JSON = global->Get(String::New("JSON"))->ToObject();
```

You could replace `"JSON"` with another property name that might be available in the global proxy object (e.g. `Int8Array`). Note that a number of objects (such as [`Int8Array`](http://bespin.cz/~ondras/html/classv8_1_1Int8Array.html)) are already provided by `v8` and there is no need to access them as above.

At this point `JSON` is also an object just like `global`. Now we don't want access a property of type object, but function. This is done by accessing the property using `Get` and casting it to a [`v8::Function`](http://izs.me/v8-docs/classv8_1_1Function.html) using `Cast` method:

```cpp
Local<Value> parse_prop = JSON->Get(String::New("parse"));
Handle<Function> JSON_parse = Handle<Function>::Cast(parse_prop);
```
Just like any other `v8::Function`, `JSON_parse` can simply be invoked using the [`Function::Call`](http://izs.me/v8-docs/classv8_1_1Function.html#ac61877494d2d8bb81fcef96003ec4059):

```cpp
std::string some_json_string = "{test: 1}";
JSON_parse->Call(JSON, 1, &some_json_string);
```
The first argument is the receiver, the second determines the number of arguments, and the third are the actual arguments.
