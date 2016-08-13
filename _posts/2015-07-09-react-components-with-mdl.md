---
layout:     post
title:      React Components with Material Design Lite
date:       2015-07-09 20:42:19
summary:    A brief introduction on how to develop React components using Google's MDL framework
categories: v8 nodejs c++
---

## Motivation
Since the very first draft of Material Design released, [it has gotten more and more attention](https://youtu.be/8UicJ0SxBwA) by the day! The main focus of Material Design was on mobile (as [delivered](http://developer.android.com/about/versions/lollipop.html#Material) with Android 5.0 Lolipop) and the only official web elements were provided by [Polymer paper elements](https://www.polymer-project.org/0.5/docs/elements/), which only supported the so-called *evergreen browsers*. So web designers and developers started writing their own material components for different frameworks ([materialup](http://www.materialup.com/) is a good place to search for various material design projects) according to the [official material design specs](https://www.google.com/design/spec/material-design/introduction.html).

Hopefully the whole chaos of different implementations comes to an end with the introduction of [Material Design Lite](http://www.getmdl.io/):

> Material Design Lite lets you add a Material Design look and feel to your websites. It doesn’t rely on any JavaScript frameworks and aims to optimize for cross-device use, gracefully degrade in older browsers, and offer an experience that is immediately accessible.

This article investigates the [latest release](https://github.com/google/material-design-lite/releases/tag/v1.0.0) of MDL and its integration with [React](http://facebook.github.io/react/).

## MDL: How it works?
MDL combines JS and CSS to realize a number of material components. Normal DOM components (such as `<button>`) can be unobtrusively *upgraded* to material components by adding specific classes to them while a central component called [`componentHandler`](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L25) keeps track of all upgraded components. A deeper description of the `componentHandler` is given in [MDL component design pattern](https://github.com/jasonmayes/mdl-component-design-pattern):

> [component handler] handles the registration of new components such that DOM upgrades are automatically performed on document load, as well as making it super easy to handle upgrades of elements that may be added after initial page load.

So as long as all DOM elements which are to be upgraded to material components are already available on `window.load`, we can be sure that `componentHandler` automatically upgrades them all. However, if you happen to dynamically add DOM elements *after* the page load, you need to [register the elements manually](http://www.getmdl.io/started/index.html#dynamic). So if your React elements are rendered after `window.load`, you might as well either use [`upgradeElement(element, jsClass)`](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L251) or [`upgradeDom(jsClass, cssClass)`](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L250). According to the documentation, these two differ in following terms:

* [`upgradeDom(jsClass, cssClass)`](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L54): Searches existing DOM for elements of our component type and upgrades them if they have not already been upgraded.

* [`upgradeElement(element, jsClass)`](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L83): Upgrades a specific element rather than all in the DOM.

in both cases `jsClass` is the *programatic name* of the material component (see below). Upon load, each component registers it self with the `componentHandler` using its constructor, programatic name, and corresponding css class. The material button for example does [the following](https://github.com/google/material-design-lite/blob/v1.0.0/material.js#L443):

```js
componentHandler.register({
  constructor: MaterialButton,
  classAsString: 'MaterialButton',
  cssClass: 'mdl-js-button'
});
```

In case of `upgradeDom`, `cssClass` is the class of DOM element which is to be upgraded to the desired material component defined by `jsClass` (i.e. its programatic name). If both parameters are absent, the whole DOM is searched for upgradable components and matching elements (i.e. those having `mdl-js-*` class) are upgraded. If `cssClass` is missing, only those upgradable components having the CSS class corresponding to `jsClass` (e.g. the `cssClass` matching `MaterialButton` is `mdl-js-button`) are upgraded. If both are provided, it is easy to figure out what happens!

In case of `upgradeElement`, `element` is the actual DOM element (e.g. `var element = document.getElementById('myElement');`). This interface requires both parameters to be present. This, however, is a restriction which is to be removed in the future (the release candidate already has solved this problem).

The list of all programatic names and corresponding CSS classes is as follows:

| Element Name   | `jsClass` | `cssClass` |
| -------------- | --------- | ---------- |
| [Button](http://www.getmdl.io/components/index.html#buttons-section) | `MaterialButton` | `mdl-js-button` |
| [Checkbox](http://www.getmdl.io/components/index.html#toggles-section) | `MaterialCheckbox` | `mdl-js-checkbox` |
| [Icon Toggle](http://www.getmdl.io/components/index.html#toggles-section/icon-toggle) | `MaterialIconToggle` | `mdl-js-icon-toggle` |
| [Menu](http://www.getmdl.io/components/index.html#menus-section) | `MaterialMenu` | `mdl-js-menu` |
| [Progress](http://www.getmdl.io/components/index.html#loading-section/progress) | `MaterialProgress` | `mdl-js-progress` |
| [Radio](http://www.getmdl.io/components/index.html#toggles-section/radio) | `MaterialRadio` | `mdl-js-radio` |
| [Slider](http://www.getmdl.io/components/index.html#sliders-section) | `MaterialSlider` | `mdl-js-slider` |
| [Spinner](http://www.getmdl.io/components/index.html#loading-section/spinner) | `MaterialSpinner` | `mdl-js-spinner` |
| [Switch](http://www.getmdl.io/components/index.html#toggles-section/switch) | `MaterialSwitch` | `mdl-js-switch` |
| [Tabs](http://www.getmdl.io/components/index.html#layout-section/tabs) | `MaterialTabs` | `mdl-js-tabs` |
| [Text field](http://www.getmdl.io/components/index.html#textfields-section) | `MaterialTextfield` | `mdl-js-textfield` |
| [Tooltip](http://www.getmdl.io/components/index.html#tooltips-section) | `MaterialTooltip` | `mdl-tooltip` |
| [Layout](http://www.getmdl.io/components/index.html#layout-section) | `MaterialLayout` | `mdl-js-layout` |
| [Data table](http://www.getmdl.io/components/index.html#tables-section) | `MaterialDataTable` | `mdl-js-data-table` |
| Ripple | `MaterialRipple` | `mdl-js-ripple-effect` |

## React + MDL
Lets say that we want to have a simple login form, which is only rendered if the session expires. Our component would look something like this:

```js
var AuthDialog = React.createClass({
  getInitialState: function() {
    return {
      loginDialogVisible: false
    };
  },
  render: function() {
    var cx = React.addons.classSet,
        classes = cx({
          "active": this.state.loginDialogVisible
        });

    var dialog = (
      <div className="login-dialog">
        <form ref="form" className="mdl-card mdl-shadow--2dp" onSubmit={this.formSubmit}>
          <div className="mdl-card__title mdl-card--expand">
            <h2 className="mdl-card__title-text">
              Please sign in to continue
            </h2>
          </div>
          <div className="mdl-card__supporting-text">
            <div className="mdl-textfield mdl-js-textfield mdl-textfield--floating-label">
              <input ref="user" className="mdl-textfield__input" type="text" id="username" />
              <label className="mdl-textfield__label" htmlFor="username">Username</label>
            </div>
            <br/>
            <div className="mdl-textfield mdl-js-textfield mdl-textfield--floating-label">
              <input ref="pass" className="mdl-textfield__input" type="password" id="password" />
              <label className="mdl-textfield__label" htmlFor="password">Password</label>
            </div>
            <br/>
            <div className="mdl-card__actions mdl-card--border">
              <button ref="submit" className="mdl-button mdl-js-button mdl-button--raised mdl-button--accent mdl-js-ripple-effect">
                Submit
              </button>
            </div>
          </div>
        </form>
      </div>
    );

    return (
      <div id="auth" className={classes}>
        { this.state.loginDialogVisible ? dialog : '' }
      </div>
    );
  },
  componentDidUpdate: function() {
    // This upgrades all upgradable components (i.e. with 'mdl-js-*' class)
    componentHandler.upgradeDom();

    // We could have done this manually for each component
    /*
     * var submitButton = this.refs.submit.getDOMNode();
     * componentHandler.upgradeElement(submitButton, "MaterialButton");
     * componentHandler.upgradeElement(submitButton, "MaterialRipple");
     */
  },
  formSubmit: function(e) {
    e.preventDefault();

    var user = this.refs.user.getDOMNode().value,
        pass = this.refs.pass.getDOMNode().value;

    // Authenticate with user/pass
    // [...]
    if (authenticated) {
      // Hide the dialog
      this.setState({loginDialogVisible: false});
    }
  },
  updateOnExpiry: function() {
    // Register timer or event to detect session expiry
    // [...]
    // Check if session is expired
    if (expired) {
      this.setState({loginDialogVisible: true});
    }
  }
});
```

Since there is no guarantee (and most probably it can be assumed) that this component is rendered after `window` has already been loaded, we need to upgrade our DOM elements to material components using the `componentHandler`. The best [lifecycle](https://facebook.github.io/react/docs/component-specs.html) of a React component to trigger the `componentHandler` is in my opinion `componentDidUpdate`, since it is:

> Invoked immediately after the component's updates are flushed to the DOM.

As it can be seen from the code above, DOM elements can be upgraded either using the `upgradeDom` or `upgradeElement`. My favorite option is to use the former (i.e. `upgradeDom`) without any parameters just to make sure every candidate is actually upgraded and to avoid upgrading each component.

## Conclusion
Integrating MDL with React is as easy as calling a single method: `componentHandler.upgradeDom();`. In my opinion, the best place to put this code is in the `componentDidUpdate` callback, which is invoked just after and element has been updated and successfully rendered. This way we can make sure that our React component already exists in the DOM and can be upgraded by the `componentHanlder`.
