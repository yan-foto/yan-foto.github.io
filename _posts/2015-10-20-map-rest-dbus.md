---
layout:     post
title:      Map an existing REST API to a D-Bus service
date:       2015-10-20 18:07:19
summary:    The journey of porting an already designed REST API to a D-Bus service while trying to stay loyal to the original design.
categories: dbus linux rest api
---

## The rationale
As I started my new job, I was appointed to design and implement a REST API through which the industrial machines of our company were to be controlled. The existing controlling software used a permanent serial connection to communicate with the machine firmware. Since the serial connection could not be shared, either only my HTTP server (handling REST queries) or the controlling software could communicate with the machine at a time. Since both the controlling software and the HTTP server were running on a Linux machine, it was only convenient to use D-Bus as our IPC tool to enable both applications control the machine at the same time.

## Disclaimer
I consider a fully one-to-one mapping of a REST API (including the HTTP server, etc.) to a D-Bus service *madness*! What follows is just a simple approach which should be regarded as a guide rather than a full specification.

## Object Path | Server URL
A REST API is provided by a server which is reachable at a specific URL. Resources are to found under specific paths. For example `http://api.quaintous.com` is where our API server resides and `http://api.quaintous.com/users` represents the `users` resource. Similarly we define a single object on D-Bus which corresponds to the API server. The interfaces of this object correspond to resources. So in accordance to our previous example the object path would be `/com/quaintous/api` and the `users` interface would have the name `com.quaintous.api.users`.

At first I considered having each resource provided by its own object (e.g. `users` on path `/com/quaintous/api/users`), soon to find out that this would only complicate the design and D-Bus interfaces provide a good enough abstraction for my goal.

## HTTP Methods | Interface members
Each REST resource responds to different HTTP methods (e.g. `GET`, `PUT`, `POST`, `DELETE`) in a different manner (e.g. `POST http://api.quaintous.com/users` creates a user). Respectively the D-Bus interfaces could also provide methods with same names (e.g. `com.quaintous.api.users.PUT`) providing the same semantics.

HTTP requests can carry parameters as [query string](https://en.wikipedia.org/wiki/Query_string) or as [message body](https://en.wikipedia.org/wiki/HTTP_message_body). On the other hand, D-Bus methods are normal functions with a number of parameters and a result value. Thus, there is no way of mapping HTTP request to D-Bus method calls without further specification. So I decided to have my interface methods all have the same signature with a string (serialized JSON) as input parameter and another one as output parameter. This way invocation of interface methods resembles an HTTP request with message body parameters. Since my API used JSON as the default media type for requests and responses. I decided to have my D-Bus parameters (both input and output) also in JSON format.

 Considering the fact that D-Bus has its own [marshaling](http://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol-marshaling) methods, it might sound irrational to have another layer of serialization (JSON strings) on the top. I could also try to be more idiomatic and use the closest thing resembling JSON in D-Bus, that is to mix and match [container types](http://dbus.freedesktop.org/doc/dbus-specification.html#container-types), to imitate an HTTP request with JSON body. However, following reasons might justify this compromise:

 * Simpler development: there is most probably a JSON library for your programming language of your desired D-Bus binding with serializing/deserializing facilities. So there is no need to understand how each language maps D-Bus types such as `STRUCT` or `DICT_ENTRY`.
 * Faster debugging: The chance that a simple method with one string as input and another one as output string is wrongly implemented is really low. The human readable format of JSON would help to pinpoint any problems very quickly.
 * JSON as established format: it is possible to forward HTTP requests received by the API server directly to the corresponding D-Bus interface without the need of any manipulation whatsoever.

### Gotchas
It is usual for REST APIs to address resources using their IDs. For example to fetch the user with ID `1` one would simply `GET` the resource at `http://api.quaintous.com/users/1`. D-Bus however imposes some restrictions on [interface names](http://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol-names): it doesn't allow interface name elements to begin with a number. So we could not use `api.quaintous.com.users.1` as interface name. I suggest prefixing IDs with underscores in interface names: `api.quaintous.com.users._1`.

## Conclusion
I this article a rather simple approach was provided to map an existing REST API to a D-Bus service. Such mapping might be useful if users/developers are already accustomed to existing procedures and data structures and want to migrate to and understand a new approach. The provided has the advantage of being easily understandable. Its downside, however, is the introduction of serializing (JSON) layer atop of D-Bus's marshaling facility.

The following table provides an overview of the mapping approach:

| REST concept | D-Bus counter part | REST example | D-Bus Example |
|--------------|--------------------|--------------|---------------|
| API base URL | Object path | `http://api.quaintous.com` | `/com/quaintous/api` |
| Resource URL | Interface | `http://api.quaintus.com/users` | `/com/quaintous/api/users` |
| HTTP method | Interface member | `GET` | `com.quaintous.api.users.GET` |
