---
layout:     post
title:      Cross-platform development with web technologies
date:       2016-05-26 10:30:19
summary:    Electron has become everybody's darling when it comes to platform independent desktop application development. It is fast and easy to develop with but it comes with a price of high resources
categories: electron cross-platform imperative declarative js html css
---

# tl;dr
Electron (Chromium + Node.js + Stereoids) is a promising platform for cross-platform desktop application using web technologies. As easy at it is to develop Electron applications, it is also easy to abuse the platform on cost of user resources. If you want a pretty little desktop container to show your web application, don't bundle a whole browser (Chromium), and JavaScript runtime environment (Node.js) for it. It is bad practice in every term of software design and architecture!

# Developing cross-platform
There is no average user when it comes to desktop applications, everything is variable from hardware to operating system, system libraries etc. So developers are keen on approaches enabling them to reach out to as many users as possible. Different programming languages take different ways from multi-plaform multi-arch compilers (e.g. `C++`) and interpreters (e.g. `Python`) to dedicated virtual machines in which the software (or a compiled version of it) runs (e.g. Java).

Business logic aside, a modern application targeting non-technical users requires a good GUI. That, exactly, is the heart of complications when it comes to developing cross-platform application. Creating a GUI imperatively, as it is the case for [Java's Swing](https://docs.oracle.com/javase/tutorial/uiswing/start/about.html) or [Python's Tkinter](https://wiki.python.org/moin/TkInter), not only takes a higher development time, it is also hard to maintain and extend. In imperative programming, everything is explicitly defined, whereas in declarative programming you focus on the *what* and not *how*. Consider the example of searching in a dataset, you could imperatively write a loop, iterate through the data and search for some specific criterion or you could use `SQL`'s `SELECT` just to declaratively denote what you want without taking care of the internals. Let's take a look at an example of showing a simple button with Java (imperative) and HTML (declarative):

```java
class ButtonDemo extends JFrame {
  JPanel() {
    // Create button
    JButton btn = new JButton();
    btn.setText('PUSH ME!');
    this.add(btn);

    // Pack and show
    this.pack();
    this.setVisible(true);
  }

  /** main method somewhere here */
}
```

```html
<!DOCTYPE html>
<html lang="en">
<body>
  <button>PUSH ME</button>
</body>
```
It is not hard to imagine the exponential growth of code base when developing GUIs imperatively comparing to a declarative approach. So let's take a look at a declarative approach which is the most familiar: the web!

# HTML/CSS/JS: Web technologies
In the early times of HTML in the mid nineties, writers of web pages started wanting to have more influence on how their pages are rendered:

> In fact, it has been a constant source of delight for me over the past year to get to continually tell hordes (literally) of people who want to -- strap yourselves in, here it comes -- control what their documents look like in ways that would be trivial in TeX, Microsoft Word, and every other common text processing environment: "Sorry, you're screwed." -- [*www-talk* mailing list (1994)](https://www.w3.org/Style/LieBos2e/history/Overview.html)

The [first draft of CSS](https://www.w3.org/People/howcome/p/cascade.html) was published in the October of the same year and made way for styling HTML files in a separate manner (even though it is still possible to have inline styling in HTML documents).

Both HTML and CSS are domain specific languages and does not support general purpose programming. This is where JavaScript comes in play with variables, loops, first-clas functions, object orientation and all the rest we know from programming 101.

## Cross-platform apps with web technologies
It is the perfect combination: HTML to define the structure, CSS to define the style and JavaScript to programmatically manipulate both HTML and CSS and an API with a number of other goodies. Developing user interfaces this way is fast, clean, and easy. However, this approach has two caveats:

 1. A browser is required to render and run our code.
 2. Browsers provide a limited (and restricted) JavaScript API (e.g. you cannot open a local file in browser)

The first item is not necessarily a handicap: browsers are ubiquitous, so the chances are high that the user has already have one installed. But it doesn't mean that those browsers are from the same vendor or of the same version. When it comes to the second one, the browsers also have an answer in terms of plugins, extension, or addons (e.g. [Chrome's `fileSystem` API](https://developer.chrome.com/apps/fileSystem) for extensions) so you can have an application which makes use of HTML/CSS/JS, runs inside the browser, and have access to an extended API. However, you wouldn't want to force your users to install a specific browser just so they can run your application.

# Enters Electron
Electron became very popular very fast partly because of [Atom editor](https://atom.io/) (which I am using to write this post):

> "Electron uses Chromium and Node.js so you can build your app with HTML, CSS, and JavaScript." -- [project page](http://electron.atom.io/)

In retrospective it seems only logical to makes use of [Chromium's](https://www.chromium.org/Home) rendering (and other) features alongside Node.js (which is based upon [Google's V8](https://developers.google.com/v8/) JavaScript runtime) to have the browser experience plus extended API and minus the restrictions to have the perfect approach to (rapid) cross-platform application development (watch [this video](https://youtu.be/_dkeD3OZ218) for more information).

Developing Electron applications is pretty straight forward for those who are familiar with web technologies. It has it's own package format that makes it easy to distribute the application and run it anywhere in Electron. You can always go further and even bundle Electron's binaries alongside your application so no other dependencies are required.

## Production-ready applications
Just like Java `.jar` files, Electron applications can be packaged in [`.asar`](https://github.com/electron/electron/blob/master/docs/tutorial/application-packaging.md) files. So if you have Electron available on your system you can use it just like a container to run your `.asar` package. It also possible to package the application with Electron to have a self-contained application which doesn't require Electron being available on the system. The former requires to have the correct version of Electron installed (as e.g. `v1.x` is backwards incompatible with `v0.3x`) and the latter shoots the package size up to 30 to 40 MB when (g)zipped. Since packaging Electron binaries alongside the application is the most convenient way for end-users, many developers also prefer this way instead of just providing the asar package. So for a simple e-mail application (e.g. Nylas N1) or collaborative notepad (e.g. Simplenote) at least 100 MB disk space is required which is a relatively huge number when compared to native apps (e.g Geary e-mail client takes 4 MB of space).

When it comes to distribution, there is no infrastructure comparable to Maven's central repository, Node's npm, Python's pip, Chrome's web store, etc. It is the developer's job to prepare the executables for different operating systems and architectures. This is also another reason why developers prefer to bundle everything together and limit the dependencies to virtually zero.

## Abusing Electron
In my opinion the abuse starts as soon as electron binaries are bundled with the actual application. As I write this post, there are 112 applications [listed](http://electron.atom.io/apps/) on Electron's official website and that is not a complete list of all Electron apps out there. Let's say that a user downloads 25 apps. Assuming each app consumes about an average of ~150 MB, the total disk space amounts to ~3.5 GB of which about 2/3 (~2 GB) is the same code base (Chromium + Node.JS). As the number of Electron instances increase, so does the number of used system resources, among which also resources which can be shared (e.g. as Chromium shares browser process with multiple isolated renderer processes). Regarding memory usage, it's [not easy](http://blog.chromium.org/2008/09/google-chrome-memory-usage-good-and-bad.html) to accurately measure how much memory is exactly used by Chromium, but exact numbers aside, this situation can also be looked at in a different manner, namely that we now have 25 browsers and 25 JavaScript runtime environments installed on our system (see previous section). Is it bad? Well you can decide for your self. One thing is for sure: it should be considered bad practice to bundle everything, even those modules that we might not need, together just to embed a web page inside a desktop container (much like what Chrome's `--app` flag does) even if does not pose a difficulty in terms of system resources in small scale. You wouldn't go ahead and statically link all the shared libraries available on your system just in case you *might* need them later.

I took the liberty of taking a look at one of rather popular Electron apps [Google Play Music Desktop Player](http://www.googleplaymusicdesktopplayer.com/) with more than [2500 stars](https://github.com/MarshallOfSound/Google-Play-Music-Desktop-Player-UNOFFICIAL-/stargazers) on GitHub. The project claims to be "more resource efficient" comparing to running Google Music's web app. The source code, however, reveals that it simply [embeds](https://github.com/MarshallOfSound/Google-Play-Music-Desktop-Player-UNOFFICIAL-/blob/v3.2.4/src/public_html/index.html#L30) the web player and introduces some JavaScript modules to tweak the functionality. Some of these extended features (e.g. custom theming) can be reached using augmented browsing (e.g. using Greasemonkey in FireFox), some by writing a browser extension/app (e.g. to support media keys), and some *might* actually require APIs that a browser does not provide. At the end of the day, I have a full fledged browser that can directly use [hardware accelerated facilities](http://www.chromium.org/developers/design-documents/gpu-accelerated-compositing-in-chrome) to rapidly render complex 3D structures just to embed a web app! The developer [counts](https://github.com/MarshallOfSound/Google-Play-Music-Desktop-Player-UNOFFICIAL-/issues/1111#event-671157520) a number of reasons to justify it being more resource efficient, whereas none of which are actually compelling in my opinion.

# Future of cross-platform
In an upcoming post I will take a look at how existing corss-platform approaches tackle the problem of heterogeneity and how some of these solution can be tailored for web technologies in desktop applications. We take a look at some existing approaches which address the issues facing cross-platfrom complications.
