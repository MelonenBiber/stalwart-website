---
title: "Throttling"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "throttle"
weight: 310
toc: true
---

## Overview

Throttling refers to limiting the rate at which requests or data are processed by a system, in order to regulate its workload and avoid overloading the system. Concurrency limiting and rate limiting are two techniques used in Stalwart SMTP to control the amount of incoming traffic, ensuring that the server operates efficiently and remains protected from malicious activity.

## Settings

Stalwart SMTP supports an unlimited number of inbound throttles, which can be dynamically configured to limit requests based on multiple variables. Throttles are defined as TOML arrays under the `session.throttle[]` keys using the following attributes:

- `concurrency`: Specifies the maximum number of concurrent requests that the throttle will allow.
- `rate`: Specifies the rate limit that the throttle will impose.
- `key`: An optional list of context variables that determine where this throttle should be applied.
- `match`: An optional rule that indicates the conditions under which this throttle should be applied.

Throttles can either include both a concurrency limit and rate limit, or just one of the two strategies.

## Concurrency limit

Concurrency limiting is the process of limiting the number of simultaneous connections from a single source. This is useful in preventing a single client from overloading the server by establishing too many connections. For example, a rule can be configured to limit the number of concurrent connections from a single IP address to prevent a denial-of-service attack.

The `session.throttle[].concurrency` attribute determines the number of concurrent connections that the throttle will allow. For example, to limit the server to accept a maximum number of 5 concurrent connections globally:

```txt
[[session.throttle]]
concurrency = 5
```

Please note that the above example will impose a global concurrency limiter, to apply a more granular limiter please refer to the [throttle groups](#groups) section below.

## Rate limit

Rate limiting is the process of limiting the number of incoming requests over a specific period of time. This is useful in preventing a single client from sending too many messages in a short amount of time, which could overload the server. For example, a rule can be configured to rate limit the number of incoming emails per minute from a single IP address or domain name to prevent a spammer from sending too many messages.

The `session.throttle[].rate` attribute determines the number of incoming requests over a period of time that the rate limiter will allow. For example, to limit the server to accept a maximum of 100 messages per seconds:

```txt
[[session.throttle]]
rate = "100/1s"
```

Please note that the above example will impose a global rate limiter, to apply a more granular limiter please refer to the [throttle groups](#groups) section below.

## Groups

The `session.throttle[].key` attribute enables the creation of throttle groups based on a combination of context variables. Available context variables are:

- `remote-ip`: The IP address of the client.
- `listener`: The identifier of the listener that received the connection.
- `helo-domain`: The domain name used in the `EHLO` command.
- `sender`: The return path specified in the `MAIL FROM` command.
- `sender-domain`: The domain component of the return path specified in the `MAIL FROM` command.
- `rcpt`: The recipient's address specified in the `RCPT TO` command.
- `rcpt-domain`: The domain component of the recipient's address specified in the `RCPT TO` command.
- `authenticated-as`: The username used for authentication.

For example, to implement a concurrency limiter per remote IP address:

```txt
[[session.throttle]]
key = ["remote-ip"]
concurrency = 5
```

And, to limit the rate at which a domain name can send messages to any given recipient to 25 messages per hour:

```txt
[[session.throttle]]
key = ["sender-domain", "rcpt"]
rate = "25/1h"
```

## Rules

Throttle rules enable the imposition of concurrency and rate limits only when a specific condition is met. These [rules](/smtp/settings/rules) can be configured using the `session.throttle[].match` attribute. For example, to impose a concurrency and rate limiter by sender only for messages coming from the IP address 10.0.0.20:

```txt
[[session.throttle]]
match = {if = "remote-ip", eq = "10.0.0.20"}
key = ["sender", "rcpt-domain"]
concurrency = 5
rate = "100/1h"
```
