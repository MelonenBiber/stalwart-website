---
title: "MAIL Stage"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "mail"
weight: 304
toc: true
---

## Overview

The `MAIL FROM` command is used to initiate an SMTP message transfer by identifying the sender of the message. After the MAIL FROM command is issued, the receiving mail server checks the validity of the sender email address and ensures that the sender is authorized to send messages on behalf of that domain. If the sender email address is deemed valid, the message transfer process continues and the next command is issued to specify the recipient email address using the SMTP `RCPT TO` command.

## Settings

Currently, the only configuration option available for the `MAIL FROM` stage is the `session.mail.script` attribute, which specifies the name of the [Sieve script](/smtp/inbound/sieve) to run after a successful `MAIL FROM` command. This can be useful, for instance, for rejecting certain senders.

Example:

```txt
[session.connect]
mail = "return_path_filter"

[sieve.scripts]
return_path_filter = '''
  require ["variables", "envelope", "reject"];

  if envelope :localpart :is "from" "known_spammer" {
      reject "We do not accept SPAM here.";
  }
'''
```


