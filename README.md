# Motivation

Sometimes, developers must integrate a blocking HTTP API, whose responses take longer than developers can handle.

It could happen for several reasons:
* The purpose of running long-running jobs wasn't taken into account during the API Design, and there is no strategy for supporting an asynchronous approach
* The processes under some microservices are heavier than expected
* The API is third-party and the developer cannot change how the API responds
* etc.

This post aims to demonstrate a POC that will turn blocking APIs into async APIs using [Kong API Gateway](https://konghq.com/products/kong-gateway).
I know there are other ways to fix this issue, but the purpose is just to explore the Kong PDK (Plugin Development Kit) and make a fast POC for proving it can be handled by Kong Gateway

## Requirements

- [x] Any request captured by the Async Plugin generates an ID and returns it in a header `X-Async-Kong-Id`
- [x] The captured request is responded with Status [`202 Accepted`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202), and the request is forwarded upstream in a different thread
- [x] When the client sends the same request adding the header `X-Async-Kong-Id` and upstream hasn't respond yet, the plugin responds `202 Accepted` as a signal that it is in progress
- [x] When upstream responds, the plugin will switch `202 Accepted` to the real response
- [x] When the client sends the same request adding the header `X-Async-Kong-Id` and upstream already responded, the plugin returns the upstream responses including headers and body

## TL;DR

Because I am just running a POC on my local machine, I used [Kong Pongo cli](https://github.com/Kong/kong-pongo).

### Timers (async)

I started the POC using `ngx.timer`, however, I wanted to go further and check an experimental Kong library [`lua-resty-timer`](https://github.com/Kong/lua-resty-timer) that schedules efficiently tasks. 
You can see more details about it [here](https://konghq.com/blog/engineering/scalable-timer-library).

Using `timer_sys:at(0.1, send_upstream_request)` I could schedule my function `send_upstream_request`.
Using this, I found a bug or a wrong description, and I needed to use `0.1` instead of `0` (immediate execution). I created a [PR here](https://github.com/Kong/lua-resty-timer-ng/pull/33).

Using `Pongo` forced me to change the docker image for installing `lua-resty-timer` via luarocks.

### Forwarding the request

I could easily use `httpc:request_uri` from `lua-resty-http`.

### Temporal storage

Because it is a POC, I used custom entities as a first approach. 
For something more serious, I would store temporal responses in a different format, and I would consider using queues or another approach that could escalate in a distributed environment.

Also, Cache can be used for binding a TTL to temporal requests/responses.

### TODO






