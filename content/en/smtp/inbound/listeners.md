---
title: "Listeners"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "listeners"
weight: 301
toc: true
---

## Listeners

Stalwart SMTP offers the ability to configure multiple listeners, which are responsible for receiving incoming SMTP connections. There is no limit to the number of listeners that can be created, and the behavior of each listener can be customized by the administrator. This means that instead of having a predetermined listener for each of the commonly used ports, such as 25 for unencrypted SMTP, 587 for SMTP submissions, and 465 for TLS encrypted SMTP submissions, Stalwart SMTP leaves it to the administrator to define the function of each listener.

## Bind settings

Listeners are set up in the configuration file using the `server.listener.<id>.bind` attribute and specifying the IP addresses and ports where the listener will receive incoming connections. A listener can be bound to multiple IP v4 or v6 addresses and ports, for example:

```txt
[server.listener."smtp"]
bind = ["10.0.0.21:25", "10.0.0.22:25", "a:3f::2"]
```

Or, to bind a listener to all interfaces:

```txt
[server.listener."smtp"]
bind = "0.0.0.0:25"
```

## Protocol

By default, incoming connections use the SMTP protocol. To configure Stalwart SMTP as an LMTP server, the `server.listener.<id>.protocol` attribute must be set to `lmtp`. For example:

```txt
[server.listener."lmtp"]
bind = ["0.0.0.0:24"]
protocol = "lmtp"
```

## TLS settings

The default settings for the server's TLS configuration can be found under the `server.tls` key and include the following options:

- `enable`: Specifies whether the listener should use TLS encryption for incoming connections. If set to `true`, the listener will require clients to connect using an encrypted connection.
- `implicit`: Specifies whether the listener should use implicit or explicit TLS encryption. If set to `false` (the default), the listener will use explicit TLS encryption, which requires clients to initiate a `STARTTLS` command before upgrading the connection to an encrypted one. If set to `true`, the listener will use implicit TLS encryption, which requires the connection to be encrypted from the start.
- `timeout`: Specifies the amount of time the listener should wait for a client to initiate the TLS handshake before timing out the connection.
- `certificate`: Specifies the name of the certificate to use for the listener. The certificate must be defined in the `certificate.<name>` section of the configuration file.
- `sni`: Specifies a list of subject names and certificates that the listener should use based on the subject name presented by the client during the TLS handshake. This is useful in scenarios where multiple hostnames are hosted on a single IP address. 
- `protocols`: Specifies the list of TLS protocols that the listener should allow clients to use.
- `ciphers`: Specifies the list of ciphers that the listener should allow clients to use. If left empty, the listener will use the default ciphers defined by the TLS library.
- `ignore-client-order`: Specifies whether the listener should ignore the order of ciphers presented by the client and use the order specified in the ciphers option. If set to `false`, the listener will use the order presented by the client.

The following example defines two SNI subjects (otherdomain.org and otherdomain.net) and specifies the TLS protocols and ciphers to use:

```txt
[server.tls]
enable = true
implicit = false
timeout = "1m"
certificate = "default"
sni = [ { subject = "otherdomain.org", certificate = "otherdomain_org" },
        { subject = "otherdomain.net", certificate = "otherdomain_net" } ]
protocols = [ "TLSv1.2", TLSv1.3" ]
ciphers = [ "TLS13_CHACHA20_POLY1305_SHA256", "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256" ]
ignore-client-order = true
```

It is possible to override the default TLS settings on a per-listener basis as detailed in the following sections.

### Certificates

TLS certificates are defined in the configuration file under the `certificate.<name>` key and require the following parameters:

- `cert`: The path or content of the TLS certificate file. This can either be directly embedded in the configuration file or referenced from an external file using the `file://` prefix.
- `private-key`: The path or content of the TLS private key file. This should not be embedded directly in the configuration file, and instead be referenced from an external file using the `file://` prefix.

The following example defines a TLS certificate named `default` with an embedded certificate and the contents of the private key read from a file:

```txt
[certificate."default"]
cert = '''-----BEGIN CERTIFICATE-----
MIIFCTCCAvGgAwIBAgIUCgHGQYUqtelbHGVSzCVwBL3fyEUwDQYJKoZIhvcNAQEL
...
0fR8+xz9kDLf8xupV+X9heyFGHSyYU2Lveaevtr2Ij3weLRgJ6LbNALoeKXk
-----END CERTIFICATE-----
'''
private-key = "file:///usr/local/stalwart-smtp/etc/private/tls.key"
```

## TCP socket options

The default TCP socket options for the server can be found under the `server.socket` key and include the following options:

- `reuse-addr:` Specifies whether the socket can be bound to an address that is already in use by another socket. Setting this option to `true` allows the socket to reuse the address and helps reduce the number of errors caused by trying to bind to an address that is already in use.
- `reuse-port`: Specifies whether multiple sockets can be bound to the same address and port. Setting this option to `true` allows multiple sockets to listen on the same port, which can be useful in some load balancing scenarios.
- `backlog`: Specifies the maximum number of incoming connections that can be pending in the backlog queue. When the number of incoming connections exceeds this value, new connections may be rejected.
- `ttl`: Specifies the time-to-live (TTL) value for the socket, which determines how many hops a packet can make before it is discarded.
- `send-buffer-size`: Specifies the size of the buffer used for sending data. Increasing the size of the buffer can help improve the performance of the socket, but also increases memory usage.
- `recv-buffer-size`: Specifies the size of the buffer used for receiving data. Increasing the size of the buffer can help improve the performance of the socket, but also increases memory usage.
- `linger`: Specifies the time to wait before closing a socket when there is still unsent data. Setting this option to a non-zero value can help ensure that all data is sent before the socket is closed.
- `tos`: Specifies the type of service (TOS) value for the socket, which determines the priority of the traffic sent through the socket.

It is possible to override the default TCP socket options on a per-listener basis as detailed in the following sections.

## Overriding defaults

Both the TLS settings and TCP socket options can be customized for each listener individually by adding the desired attribute to the `server.listener.<id>` key of that specific listener. For example, to override implicit TLS for a listener:

```txt
[server.listener."submissions"]
tls.implicit = true

[server.tls]
implicit = false
```

Or, to increase the backlog for a certain listener:

```txt
[server.listener."smtp"]
socket.backlog = 2048

[server.socket]
backlog = 1024
```

## Example

The following example defines an SMTP listener on port `25`, an SMTP submissions listener on port `587` and a secure SMTP submissions listener on port `465` with implicit TLS enabled:

```txt
[server.listener."smtp"]
bind = ["0.0.0.0:25"]

[server.listener."submission"]
bind = ["0.0.0.0:587"]

[server.listener."submissions"]
bind = ["0.0.0.0:465"]
tls.implicit = true

[server.listener."management"]
bind = ["127.0.0.1:8686"]
protocol = "http"
```

