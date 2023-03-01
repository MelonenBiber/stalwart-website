---
title: "Sieve scripting"
description: ""
date: 2022-08-28T12:49:20Z
lastmod: 2022-08-28T12:49:20Z
draft: false
images: []
type: docs
menu:
  smtp:
    parent: "inbound"
    identifier: "sieve"
weight: 309
toc: true
---

## Overview

Sieve is a scripting language used to filter and modify email messages. It provides a flexible and powerful way to manage email messages by automatically filtering, sorting, and transforming them based on a wide range of criteria.  Rather than relying on a proprietary DSL, Stalwart SMTP uses Sieve as its default scripting language primarily because it is sufficiently powerful to handle most filtering tasks and is an established [internet standard](https://www.rfc-editor.org/rfc/rfc5228.html).

A typical Sieve script consists of one or more rules, each consisting of a test and an action. The test checks a specific attribute of the message, such as the sender's address or the subject line, while the action specifies what to do with the message if it matches the test. This manual does not cover how to write Sieve scripts but tutorials and examples can be found at [https://sieve.info](https://sieve.info).

## Scripts

Sieve scripts are specified under the `sieve.scripts.<name>` key and can be invoked directly from any of the stages of an SMTP transaction or imported from other scripts using the `include` command. In the configuration file, Sieve scripts can be either embedded as text or loaded from external files using a `file://` URL, for example:

```txt
[sieve.scripts]
script_one = '''
    require ["variables", "extlists", "reject"];

    if string :list "${env.helo_domain}" "list/blocked-domains" {
        reject "551 5.1.1 Your domain '${env.helo_domain}' has been blacklisted.";
    }
'''
script_two = "file:///usr/local/stalwart-smtp/etc/sieve/my-script.sieve"
```

## Context Variables

Sieve scripts executed in Stalwart SMTP have access to the following environment variables:

- `env.remote_ip`: The email client's IP address.
- `env.helo_domain`: The domain name used in the EHLO/LHLO command.
- `env.authenticated_as`: The account name used for authentication.

The envelope contents can be accessed using the [envelope test](https://www.rfc-editor.org/rfc/rfc5228.html#page-27) or through variables:

- `envelope.from`: The return path specified in the `MAIL FROM` command.
- `envelope.to`: The recipient address specified in the last `RCPT TO` command.
- `envelope.notify`: The `NOTIFY` extension parameters.
- `envelope.orcpt`: The `ORCPT` value specified with the DSN extension.
- `envelope.ret`: The `RET` value specified with the DSN extension.
- `envelope.envid`: The `ENVID` value specified with the DSN extension.
- `envelope.by_time_absolute`: The specified absolute time in the `DELIVERBY` extension.
- `envelope.by_time_relative`: The specified relative time in the `DELIVERBY` extension.
- `envelope.by_mode`: The mode specified with the `DELIVERBY` extension.
- `envelope.by_trace`: The trace settings specified with the `DELIVERBY` extension.

## Runtime Settings

Stalwart SMTP compiles all defined Sieve scripts when it starts and executes them on demand using the Sieve runtime. The runtime is configured with the following parameters which are available under the `sieve` key:

- `from-name`: Defines the default name to use for the from field in email notifications sent from a Sieve script.
- `from-addr`: Defines the default email address to use for the from field in email notifications sent from a Sieve script.
- `return-path`: Defines the default return path to use in email notifications sent from a Sieve script.
- `sign`: Lists the [DKIM](/smtp/auth/dkim) signatures to add to email notifications sent from a Sieve script.
- `hostname`: Sets the local hostname to use when generating a `Message-Id` header. If no value is set, the `server.hostname` value is used instead.
- `user-database`: Specifies the database to use when running `execute :query` Sieve commands.
- `limits.redirects`: Specifies the maximum number of `redirect` commands that a Sieve script can execute.
- `limits.out-messages`: Specifies the maximum number of outgoing email messages that a Sieve script is allowed to send.
- `limits.received-headers`: Specifies the maximum number of `Received` headers that a message can contain.
- `limits.cpu`: Specifies the maximum number of instructions that a Sieve script can execute.
- `limits.nested-includes`: Specifies the maximum number of nested includes that a script can perform.
- `limits.duplicate-expiry`: Specifies the default expiration time for the expiry Sieve test.


Example:

```txt
[sieve]
from-name = "Automated Message"
from-addr = "no-reply@foobar.org"
return-path = ""
hostname = "mx.foobar.org"
sign = ["rsa"]
use-database = "sql"

[sieve.limits]
redirects = 3
out-messages = 5
received-headers = 50
cpu = 10000
nested-includes = 5
duplicate-expiry = "7d"
```

## Execute extension

Stalwart SMTP adds to the Sieve language the `vnd.stalwart.execute` extension which allows scripts to execute external commands or SQL queries using the `execute` command (and test). The `execute` command/test expects as arguments the command or query name followed by the command arguments or query values.

### SQL queries

The `:query` tag is used to execute an SQL query on the server defined in the `sieve.use-database` attribute, for example:

```sieve
require ["vnd.stalwart.execute", "variables", "envelope", "reject"];

set "triplet" "${env.remote_ip}.${envelope.from}.${envelope.to}";

if not execute :query "SELECT EXISTS(SELECT 1 FROM greylist WHERE addr=? LIMIT 1)" ["${triplet}"] {
    execute :query "INSERT INTO greylist (addr) VALUES (?)" ["${triplet}"];
    reject "422 4.2.2 Greylisted, please try again in a few moments.";
}
```

### External programs

The `:binary` tag is used to execute external binaries, for example:

```sieve
require "vnd.stalwart.execute";

if execute :binary "/usr/local/stalwart/bin/validate.sh" ["${env.remote_ip}", "${envelope.from}"] {
    reject "You are not allowed to send e-mails.";
}

```

## Supported extensions

The Sieve extensions supported in Stalwart SMTP are listed in the [RFCs conformed section](/smtp/development/rfc/).
