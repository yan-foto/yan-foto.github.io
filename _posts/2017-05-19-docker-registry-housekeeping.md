---
layout:     post
title:      Housekeeping Docker registries using deckschrubber
date:       2017-05-19 6:30:19
summary:    In contrast to Docker hub/store, hosted Docker registries tend to have limited space. This article shows how to clean old/outdated images from your own registry.
categories: docker registry golang
---

# Introduction
I use Docker for my CI pipeline: testing, building, etc. Everything that is built successfully, is packed as a Docker image and is pushed on a private [Docker registry](https://docs.docker.com/registry/). Before long I was having space issues (which turned out to **not** to be caused by the registry) and decided to purge all images older than 3 months.

I decided on a simple program with the following requirements:

  1. It should be a single binary with no dependencies
  2. It should work with remote (non-local) registries
  3. It should conform to Docker APIs (no dirty hacking!)
  4. It should be possible to remove images by age
  5. It should *always* keep the latest image

Requirement 1. and 2. make sure that no other dependencies/runtime env is needed and the housekeeping can be done remotely. Requirement 3. makes sure that the communication procedure with the registry is undesrstood correctly! Requirement 4. allows fine-grain filtering. Requirement 5. makes sure that in the worst case, at least one image per repository stays on the server.

There are similar projects implemented in python, ruby, etc. As I wanted to have the housekeeping locally on my Docker registry server, I deemed it utmost unnecesary to have python, etc. installed on my production server just to remove old Docker images from the registry. And that's how [`deckschrubber`](https://github.com/fraunhoferfokus/deckschrubber) was born.

# Implementation
I chose [Go](https://golang.org/) as it allows building cross platform executable (also see [Go CGO](https://golang.org/cmd/cgo/)) and as Docker registry itself is implemented in Go (Req. 1.). Go packages from [Docker registry](https://github.com/docker/distribution) are used to make sure the program conforms to the API (Req. 2. and 3.).

Filtering (Req. 4 und 5.) is done as follows:

  1. The list of repositories is fetched from the registry (e.g. from `http://localhost:5000`)
  2. The list of images of each repository is fetched
  3. Blob details for each image is fetched (this contains creation date)
  4. A sorted (by date) list of images for each repository is created
  4. For each respository, images older than the given age (except latest image) are removed

**NOTE:** to have the deletion take effect immediately, the garbage collection of the registry has to be triggered:

```bash
registry garbage-collect  /etc/docker/registry/config.yml
```

It is assumed that the configuration for the registry is available under the default path `/etc/docker/registry/config.yml`.

# Usage

A pre-built binary for linux x64 systems is [released](https://github.com/fraunhoferfokus/deckschrubber/releases) with every version of the project on its [GitHub page](https://github.com/fraunhoferfokus/deckschrubber). Otherwise, it can be built directly from the source (given that Go is installed):

```bash
git clone https://github.com/fraunhoferfokus/deckschrubber.git
cd deckschrubber
go get ./...
go install
$GOPATH/bin/deckschrubber -registry http://localhost:5000 -repos 25 -day 14 -dry
```

For details on arguments and more examples, checkout the [project page](https://github.com/fraunhoferfokus/deckschrubber#arguments).