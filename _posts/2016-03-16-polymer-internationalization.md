---
layout:     post
title:      Polymer 1.0 Internationalization
date:       2016-03-17 14:27:19
summary:    A simple component which provides other components with a localization behavior which can be either used for computed bindings or simply anywhere in code.
categories: polymer internationalization i18n localization custom-elements web-components
---

## Introduction
Let's face it: internationalization is inevitable! As the importance of dynamic, client-side rendered web components increases, so does the need for a client-side i18n solution. In this article a simple component is introduced which asynchronously loads a translation file (in JSON) and provides other components with an `I18N` [behavior](https://www.polymer-project.org/1.0/docs/devguide/behaviors.html) to be used for computed bindings or simply directly in code.

## tl;dr
Grab the code on [gitlab](https://gitlab.com/quaintous-polymer/quaintous-i18n.git) or install it using Bower:

```bash
bower install https://gitlab.com/quaintous-polymer/quaintous-i18n.git
```

now you can start using it in a custom element:

```html
<link rel="import" href="../../components/polymer/polymer.html">

<dom-module is="localized-tag">
  <template><span>[[__(myProperty)]]</span></template>
  <script>
  Polymer({
    is: 'localized-tag',

    properties: {
      myProperty: String
    },

    behaviors: [I18N]
  });
  </script>
</dom-module>
```

note the `behaviors: [I18N]` line. Now this:

```html
<quaintous-i18n locales-path="/locales/fa.json"></quaintous-i18n>
<localized-tag my-property="hello"></localized-tag>
```

renders to:

```html
<span>سلام</span>
```

## Considerations
The previous example assumes that fetching the translation file (i.e. `fa.json`) is succeeded *before* the `localized-tag` is ready and attached, otherwise `myProperty` cannot be bound (i.e. translated) correctly when the element is rendered. This can be solved using [**conditional templates**](https://www.polymer-project.org/1.0/docs/devguide/templates.html#dom-if) as shown bellow:

```html
<template is="dom-bind">
  <quaintous-i18n locales-path="/locales/fa.json" loading="{{i18nLoading}}"></quaintous-i18n>
  <template is="dom-if" if="{{!i18nLoading}}">
    <localized-tag my-property="hello"></localized-tag>
  </template>
</template>
```

## The Element
Why create an element at all when a simple behavior (which is a simple JS object) suffies? Well, I just wanted to comply with Polymer's motto "there's an element for that". After all I believe that an element is the more elegant and readable way to do this task.

Our element depends on [`iron-ajax`](https://elements.polymer-project.org/elements/iron-ajax) and exposes a simple API with two properties `locales-path`, denoting the translation file URL, and `loading` which is `true` while the AJAX request is on the fly.

When the element is imported it registers `I18N` object in the global context (i.e. `window`) which is to be used by other Polymer elements as a behavior. After the element is loaded and attached, the `iron-ajax` loads the translations files and on success it saves the received data also in the global context in `__i18n__` variable. From now on `I18N.__(key)` can be used.

## Conclusion
There are a number of other approaches all with the goal of providing internationalization for Polymer components. This approach follows Polymer's "there's an element for that" motto and introduces an element which loads translation files asynchronously and provides other elements with a behavior which can be used for translation.
