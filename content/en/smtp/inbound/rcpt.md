---
title: "RCPT Stage"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "rcpt"
weight: 305
toc: true
---

## Overview

The `RCPT TO` command is a used to specify the recipient of an email message during the SMTP message transfer. It is used in conjunction with the `MAIL FROM` command, which specifies the sender of the email. After the `RCPT TO` command is issued, the SMTP server responds with a status code indicating whether the recipient is valid and whether it is willing to accept the email message for that recipient. If the recipient is valid and the server is willing to accept the email, the SMTP client can then proceed to send the email message using the `DATA` command.

## Settings

Under the `session.rcpt` key in the configuration file, the following attributes control the behavior of the `RCPT` command:

- `relay`: Specifies whether the SMTP server should relay messages for non-local domain names. This attribute is useful when configured as a [rule](/smtp/settings/rules) that only allows relaying for authenticated users.
- `max-recipients`: Specifies the maximum number of recipients allowed per message.
- `script`: Specifies the [Sieve script](/smtp/inbound/sieve) to execute after a successful `RCPT TO` command, which can be used to implement greylisting, among other things.
- `lookup.domains`: Specifies the lookup path for determining whether a domain is local.
- `lookup.addresses`: Specifies the lookup path for determining whether a local recipient exists.
- `lookup.vrfy`: Specifies the lookup path for verifying a recipient's address during a `VRFY` command, or set to `false` to disable address verification.
- `lookup.expn`: Specifies the lookup path for expanding a mailing list address during an `EXPN` command, or set to `false` to disable address expansion.
- `rcpt.errors.total`: Specifies the maximum number of invalid recipients allowed before a session is disconnected.
- `rcpt.errors.wait`: Specifies the amount of time to wait when an invalid recipient is received.

Example:

```txt
[session.rcpt]
script = [ { if = "authenticated-as", eq = "", then = "greylist" }, 
           { else = false } ]
relay = [ { if = "authenticated-as", ne = "", then = true }, 
          { else = false } ]
max-recipients = 25

[session.rcpt.lookup]
domains = "list/domains"
addresses = "remote/lmtp"
vrfy = [ { if = "authenticated-as", ne = "", then = "remote/lmtp" }, 
         { else = false } ]
expn = [ { if = "authenticated-as", ne = "", then = "remote/lmtp" }, 
         { else = false } ]

[session.rcpt.errors]
total = 5
wait = "5s"

[list]
domains = ["foobar.com", "foobar.net"]

[sieve.scripts]
greylist = '''
    require ["variables", "vnd.stalwart.execute", "envelope", "reject"];

    set "triplet" "${env.remote_ip}.${envelope.from}.${envelope.to}";

    if not execute :query "SELECT EXISTS(SELECT 1 FROM greylist WHERE addr=? LIMIT 1)" ["${triplet}"] {
        execute :query "INSERT INTO greylist (addr) VALUES (?)" ["${triplet}"];
        reject "422 4.2.2 Greylisted, please try again in a few moments.";
    }
'''

```