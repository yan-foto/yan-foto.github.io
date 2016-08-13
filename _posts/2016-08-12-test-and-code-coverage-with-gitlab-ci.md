---
layout:     post
title:      Test and Code Coverage with GitLab Docker Runners
date:       2016-08-12 16:30:19
summary:    This article discusses how to use Docker containers as GitLab Runners to do test and code coverage for Java and Node.js projects.
categories: gitlab ci docker test code-coverage nodejs java
---

# Introduction
GitLab allows building complex pipelines for building, testing, packaging, etc. projects using [Runners](http://docs.gitlab.com/ce/ci/runners/README.html):

>  isolated (virtual) machine[s] that picks up builds through the coordinator API of GitLab CI.

There are two types of Runners at the moment: shared and specific. Shared runners are provided by GitLab and can be used by everyone. Shared Runners, in contrast, are *private* Runners running on own infrastructure. Runners can also be utilized in various ways (shell, docker, etc.). This article is limited to shared Docker Runners.

# How is it done?
The procedure of registering and enabling Runners for a project is fairly easy:

  1. Enable a shared Runner tagged with docker under `Project > Runners`
  2. Specify a regular expression under `Project > CI/CD Pipelines > Test coverage parsing` to filter coverage from console output
  3. Create a docker image containing everything needed to build and test your project
  4. Create `.gitlab-ci.yml` in your project’s root and define how testing/coverage is to be done

In the following steps `2`, `3`, and `4` are discussed for Java and Node.js projects.

# Java

## Docker image
The docker image that I use for my projects uses [Maven](https://maven.apache.org/) for building/testing and [JaCoCo](http://www.jacoco.org/) for code coverage reporting:

```dockerfile
FROM ubuntu:16.04

MAINTAINER "Yan Foto"
LABEL version="0.1.0"

RUN apt-get update && apt-get install -y openjdk-8-jdk-headless maven libxml2-utils

CMD ["java"]
```

This image must be either on Docker hub or any other registry accessible by GitLab. For the sake of simplicity, let’s say that the image is available in Docker registry as `quaintous/java-ci`.

*Wait a minute...*: I will explain later why `libxml2-utils` is needed.

## CI Script
When using a Docker Runner, the CI script is as easy as it gets. You need to tell GitLab which image is to be pulled for build/test/coverage procedure and provide which scripts should be run after the repository is fetched:

```yml
image: quaintous/java-ci

test_coverage:
  script:
    - mvn test
    - "COVERAGE=$(xmllint --html --xpath '//table[@id=\"coveragetable\"]/tfoot//td[@class=\"ctr2\"][1]/text()' target/site/jacoco/index.html)"
    - 'echo "Coverage: $COVERAGE"
```

`mvn test` is self explanatory! The next two lines are required to parse and print the generated test coverage files by JaCoCo. The corresponding RegExp to filter out coverage results (step `2` of tl;dr) is:

```regexp
^Coverage+:\s(\d+(?:\.\d+)?%)
```

You have to set it in your `Project > CI/CD Pipelines > Test coverage parsing` settings.

# Node.js

## Docker image

The Docker image For my Node.js projects contains [Mocha](http://mochajs.org/) for testing, [Istanbul](https://github.com/gotwarlost/istanbul) for code coverage, and [standard JS](https://github.com/feross/standard) for code style:

```dockerfile
FROM node:6-slim

MAINTAINER "Yan Foto"
LABEL version="0.1.0"

RUN npm install -g standard mocha istanbul

CMD ["node"]
```

Lets say that this image is also available in a public registry as `quaintous/node-ci`.

## CI Script
Similar to the CI script, the steps are the same, except that the Runner is instructed to install all dependencies before running the test:

```yml
image: quaintous/node-ci

before_script:
  - npm install

test_coverage:
  script:
    - standard
    - istanbul cover _mocha
```

The regular expression for GitLab to parse the coverage percent (for statements) is:

```regexp
^Statements\s+:\s(\d+(?:\.\d+)?%)
```

# One last word

I didn’t want to write this article: everytime that I figour out something, it appears to me that the solution have always been easy and even trivial. However, I saw this [article](https://medium.com/@kaiwinter/javafx-and-code-coverage-on-gitlab-ci-29c690e03fd6#.whsa8wcof) and thought there might be more people like me struggling to get started with GitLab Runners.
