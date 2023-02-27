---
title: "DATA Stage"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "data"
weight: 306
toc: true
---

## Overview

The `DATA` command is used to initiate the data transfer phase of the email delivery process. Once the sender of the email has successfully completed the `MAIL FROM` and `RCPT TO` commands, they can issue the `DATA` (or `BDAT` when chunking is used) command to begin transmitting the email message. Once the entire email message has been transmitted, the SMTP server will respond with a status code indicating whether the message was accepted or rejected for delivery.

## Filtering with Sieve

Once a message has been submitted either with the `DATA` or `BDAT` command, it is possible to run a [Sieve script](/smtp/inbound/sieve) that accepts, rejects or modifies the message's contents. The `session.data.script` attribute in the configuration file specifies the name of the Sieve script to be executed.

Example:

```txt
[session.data]
script = "data"

[sieve.scripts]
data = '''
    require ["envelope", "variables", "replace", "mime", "foreverypart", "editheader", "extracttext"];

    if envelope :domain :is "to" "foobar.net" {
        set "counter" "a";
        foreverypart {
            if header :mime :contenttype "content-type" "text/html" {
                extracttext :upper "text_content";
                replace "${text_content}";
            }
            set :length "part_num" "${counter}";
            addheader :last "X-Part-Number" "${part_num}";
            set "counter" "${counter}a";
        }
    }
'''
```

## Message Limits

Stalwart SMTP server can be configured to limit messages based on their size and total number. Additionally, it supports setting a maximum limit on the number of `Received` headers, which helps to prevent message loops. These configuration attributes can be found under the `session.data.limits` key in the configuration file and include the following:

- `message`: Limits the maximum number of messages that can be submitted per SMTP session.
- `size`: Restricts the maximum size of a message in bytes.
- `received-headers`: Limits the maximum number of `Received` headers that a message can contain.

Example:

```txt
[session.data.limits]
messages = 10
size = 104857600
received-headers = 50
```

## Headers

The following attributes under the `session.data.add-headers` key determine which headers should be added to an incoming message:

- `received`: Add a `Received` header to the message, which includes the client IP address and TLS encryption information.
- `received-spf`: Add an `SPF-Received` header to the message, containing SPF authentication details.
- `auth-results`: Add an `Authentication-Results` header to the message, which includes DMARC, DKIM, SPF, ARC, and iprev authentication results.
- `message-id`: If the message does not already contain a `Message-ID` header, add one with a unique message ID.
- `date`: If the message does not already contain a `Date` header, add one with the current date.
- `return-path`: Add a `Return-Path` header to the message, which contains the address specified in the `MAIL FROM` command.

Example:

```txt
[session.data.add-headers]
received = [ { if = "listener", eq = "smtp", then = true }, 
             { else = false } ]
received-spf = [ { if = "listener", eq = "smtp", then = true }, 
                 { else = false } ]
auth-results = [ { if = "listener", eq = "smtp", then = true }, 
                 { else = false } ]
message-id = [ { if = "listener", eq = "smtp", then = false }, 
               { else = true } ]
date = [ { if = "listener", eq = "smtp", then = false }, 
         { else = true } ]
return-path = false
```

## Content filters

Content filters are external programs such as SPAM filters or Antivirus programs that analyze and modify the message contents. These filters take the message contents from the standard input (`stdin`) and send back the modified contents via standard output (`stdout`). Content filters are defined under the `session.data.pipe.<id>` key with the following attributes:

- `command`: This specifies the path to the binary program to be executed.
- `arguments`: This attribute sets the arguments to be passed to the binary program.
- `timeout`: The time to wait for the content filter to complete. If the filter takes longer than the specified time, the program is terminated and the message is delivered without modification.

Currently, Stalwart SMTP only supports external executable files for content filters, but future versions will also include support for [WASM](https://en.wikipedia.org/wiki/WebAssembly) filters.

Example:

```txt
[session.data.pipe."spam-assassin"]
command = "spamc"
arguments = []
timeout = "10s"
```

