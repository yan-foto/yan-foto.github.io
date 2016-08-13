---
layout:     post
title:      Sharing Models on Client and Server Side
date:       2015-06-07 18:40:19
summary:    See how it is possible to share MVC models between backend and frontend using JavaScript on both ends.
categories: mvc nodejs javascript
---

Back in the days as I wasn't a [Node.js](https://nodejs.org/) developer yet, we used to have our MVC models implemented in a server side language such as Ruby (as in [Ruby on Rails](http://rubyonrails.org/)) and once again on the client side in JavaScript. In our newest project as we decided to use Node for our backend and was discussing on wether to use [AngularJS](https://angularjs.org/) or [React](https://facebook.github.io/react/) for the frontend, I voted for React because I thought we don't need to implement yet another MVC system (such as Angular) on the frontend. The rationale was to share the models on both server and client side and use a lightweight library (such as React) just for the view.

In the following I suggest a simple approach enabling the models to be shared both on the server and client side. The only assumption is that Node is used in the backend and JavaScript in the frontend.

## Models

The models are inspired by Rails' [Active Record Models](http://guides.rubyonrails.org/active_record_basics.html). Each model contains a list of properties and a set of validators. The basic stub (without specific implementation) is as follows:

{% highlight coffeescript %}
class BaseModel
  # Default properties values
  default: {}
  # Set of validators
  validators: {}

  constructor: (@data) ->

  validates: (property, validator) ->

  set: (property, value) ->

  serialize: ->
{% endhighlight %}

Upon initiation the `default` values merge with `data` and from now on `data` is the object representing the __serialized__ version of the model and which can be exchanged between client and server as a [JSON](http://www.json.org/) object.

`validates` can be used in the constructor of any subclassing model to attach validators to specific properties. Next section elaborates on the concept of validators.

## Validators

A class implementing the `isValid(value) -> boolean` is considered a validator:

{% highlight coffeescript %}
class BaseValidator
  isValid: (value) ->
{% endhighlight %}

Validators can be attached to properties of a model and are invoked upon setting the value of that property by calling the `set` method of the model. The `BaseModel` maintains a mapping of property name to an array of validators.

The advantage of embedding validators inside a model is that you can use it both on server side (e.g. just before persisting data in a database) and the front end (e.g. just before submitting a form).

## Serializing and Deserializing

Since the data exchange between the frontend and backend is succeeded in JSON format and both ends are written in JavaScript, the (de)serializing is done at little to no cost.

As mentioned earlier the `data` object of any model corresponds to the serialized version of that model. Deserializing simply consists of instantiating an object of that model with the serialized `data`.

The `serialize` method of the `BaseClass` can be overwritten if property values need to be manipulated before transmission.

## An Example

Consider we want to model a nuclear plant:

{% highlight coffeescript %}
class NuclearPlantModel extends BaseModel
  default: {
    exploded: false,
    temperature: 345,
    # ...
  }

  serialize: ->
    # converting from celsius to kelvin
    data.temperature = data.temperature + 273.15
    data

  constructor: (@data) ->
    super(@data)
    validates: 'exploded', new BooleanValidator
    validates: 'temperature', new NumberValidator

{% endhighlight %}

The `BooleanValidator` can be something like:

{% highlight coffeescript %}
class BooleanValidator extends BaseValidator
  isValid: (value) ->
    $.inArray(value, [true, false, 'true', 'false', 0, 1])

{% endhighlight %}
