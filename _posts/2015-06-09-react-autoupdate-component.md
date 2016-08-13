---
layout:     post
title:      React Auto Update Component
date:       2015-06-09 19:01:19
summary:    How to build a simple that can be embedded into any React component to automatically update its state using AJAX.
categories: react mvc
---

So I was assigned to create a fairly simple dashboard with multiple [React](https://facebook.github.io/react/) components which retrieve their information over an API and update correspondingly. Since all my components used models with a similar structure (as explained in [the previous post]({% post_url 2015-06-07-sharing-models-on-client-and-server-side %})), it was fairly easy to create a straight forward element to update them.

## Auto Update Component

The auto update component has the following stub:

{% highlight javascript %}
var AutoUpdate = React.createClass({
  getDefaultProps: function() {
    return {
      url: null,
      interval: 3000,
      callback: null
    };
  },
  fetchData: function(url) {
    $.ajax({
      url: this.props.url,
      success: function(data) {
        this.props.callback(data);
      }.bind(this)
    });
  },
  componentDidMount: function () {
    setInterval(this.fetchData, parseInt(this.props.interval));
  },
  render: function() {
    return (<div className="loader" />);
  }
});
{% endhighlight %}

Of course you could do loads of fancy stuff to the component (e.g. error handling, etc), but its good enough for now! Let's see what the props are:

* `url`: API endpoint which returns the data.
* `interval`: Interval in milliseconds to refresh data.
* `callback`: A callback to be invoked as soon as the data has been fetched.

## Integration

Inside any component you could now just embed the auto update component to take care of server queries:

{% highlight javascript %}
var MyComponent = React.createClass({
  render: function() {
    return (
      <AutoComplete url="my.api/v1/me" interval="1000" callback={this.updateMe} />
      // ...
    );
  },
  updateMe: function(data) {
    // Magic!
    // this.setState(...)
  }
});
{% endhighlight %}

## Update State Automatically

You could go one step further and update the parent component's state by providing it as the callback assuming that the data provided from the API server correspond to your component's state. Consider that the API returns the following:

{% highlight javascript %}
{
  "x": 1,
  "y": 2,
  "z": 3
}
{% endhighlight %}

You can bind your components `setState` function directly to the auto update component:

{% highlight javascript %}
var MyComponent = React.createClass({
  getInitialState: function() {
    return {
        x: 0,
        y: 0,
        z: 0
    };
  },
  render: function() {
    return (
      <AutoComplete url="my.api/v1/me" interval="1000" callback={this.setState.bind(this)} />
      // ...
    );
  }
});
{% endhighlight %}

*Note* that you need to `bind` the `setState` so that it internally can access the component and not the auto update component!
