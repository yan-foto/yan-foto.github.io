---
layout:     post
title:      Linking in node-gyp For Non-C++ Programmers
date:       2015-06-12 19:01:19
summary:    Creating native Node.js addons can be real headache if you are comming from a interpreter language background. Most common pitfalls and mistakes leading to compilation and linking failures are discussed here.
categories: nodejs addon c++
---

## Introduction to `node-gyp`

`node-gyp` is the recommended tool by *Node.js* to build native (i.e. `C++`) [addons](https://nodejs.org/api/addons.html). "It bundles the gyp project used by the Chromium team and takes away the pain of dealing with the various differences in build platforms"[¹](#resources). [`gyp`](https://chromium.googlesource.com/external/gyp/+/master/docs/UserDocumentation.md) is just another build tool (like [`cmake`](http://www.cmake.org/)) which makes compiling and linking much easier. `node-gyp` by default generates a shared `.so` or `.dll` library, which can be required (e.g. using [`bindings`](https://www.npmjs.com/package/bindings)) and used by Node.js modules.

## Motivation
In my latest project, I was responsible for writing a JavaScript binding (a Node.js addon to be more specific) to an existing [hardware abstraction](https://en.wikipedia.org/wiki/Hardware_abstraction) module which was written in `C++`. This module in turn is dependent on other in-house libraries. Since all of our `C++` projects are built using `cmake` (equipped with some fancy customized macros), I was already familiar how to tell the compiler and linker where to search for sources and required libraries, and how to create an executable or libraries which knows where to look for dependencies at the runtime. All I had to do now was to translate my `cmake` knowledge to `node-gyp`.

 If someone doesn't have a solid background in `C++`, one will be confronted (sooner or later) with one of the following errors during the build process:

* `No such file or directory`
* `Unable to find libxxx.so.x`
* `Undefined symbol`
* `...`

Such errors are either compilation or linking[²](#resources) errors which we don't have in case of interpreted languages[³](#resources) (e.g JavaScript) and might be confusing to developers of such programming languages.

## Crash Course: `C++` Life Cycle

A typical `C++` program is made up of source files containing a number of `#include`s, `#define`s, and other directives. The first step in the course of making an executable or a library from source files is called **preprocessing** which takes care of the directives. The resulting file (without any directives) is fed to the **compiler** and and an object file is returned: still neither an executable nor a library. At this stage **linker** comes into action and creates a library or an executable from the object file(s). From now on the generated executable can be run or the generated library can be used by other libraries and executables.

## Configuring `node-gyp`
`node-gyp` (just like `gyp`) uses a `.gyp` file called `binding.gyp` which is used to generate a [`Makefile`](https://en.wikipedia.org/wiki/Makefile) (in Unix) or [`vcxproj`](http://blogs.msdn.com/b/visualstudio/archive/2010/05/14/a-guide-to-vcxproj-and-props-file-structure.aspx) file (in Windows) which simply denote how the native addon is to be compiled and linked. This section focuses on the structure of `binding.gyp` in case of special needs. Before continuing it is recommended to take a look at [skeleton of a `gyp` file](https://chromium.googlesource.com/external/gyp/+/master/docs/UserDocumentation.md#Skeleton-of-a-typical-Chromium-gyp-file).

As long as you don't use third party libraries, that is everything except own code, node library, or system libraries, you can follow the [official tutorial](https://nodejs.org/api/addons.html) and you're good to go. The problem begins when you want to use libraries which **are not** on the prerprocessor, or linker search paths. In former case you have to define the the path for all included headers and in the latter you have to define the path on which all used libraries reside.

* **Header files**:
`include_dirs` is an array that can be used to specify the location of all used headers.
* **Libraries** : both at the time of linking, i.e. last phase of creating a library/executable, and at runtime, i.e. using that library/executable, required libraries have to be available. In both cases `link_settings` dictionary is to be used:
  * To provide a list of required libraries to the linker, `libraries` array is used. Here you can either have paths to specific libraries or you could use `-l` and `-L`[⁴](#resources) flags to respectively define the library name and the directory to look for libraries into (example follows).
  * To introduce a "runtime search path", `rpath`[⁵](#resources) can be used inside `ldflags` array.

## Example of `binding.gyp`
In the following example it is assumed that [`Boost`](http://www.boost.org/) libraries are required for the addon, **but** these are not on the standard system paths for headers and libraries and rather reside on a custom route called `boost_root`.

{% highlight json %}
{
  "variables": {
    "boost_root%": "/path/to/boost"
  },
  "targets": [
    {
      "target_name": "fancy_addon",
      "sources": [ "addon.cpp", "fancy.cpp" ],
      "include_dirs": [
        "<@(boost_root)",
      ],
      "link_settings": {
        "libraries": [
          "-lboost_program_options",
          "-lboost_log",
        ],
        "ldflags": [
            "-L<@(boost_root)/stage/lib",
            "-Wl,-rpath,<@(boost_root)/stage/lib",
        ]
      },
      "cflags": [
        "-std=c++11"
      ],
      "cflags_cc!": [
        "-fno-rtti",
        "-fno-exceptions"
      ]
    }
  ]
}
{% endhighlight %}

## Resources

¹ [`node-gyp` Documentation](https://github.com/TooTallNate/node-gyp)

² [C++ Compilation and Linking](http://stackoverflow.com/a/6264256/2295964)

³ [Compiled vs. Interpreted](http://stackoverflow.com/a/3265602/2295964)

⁴ [`gcc` `-L` / `-l` option flags](http://www.rapidtables.com/code/linux/gcc/gcc-l.htm)

⁵ [Wikipedia `rpath`](https://en.wikipedia.org/wiki/Rpath)
