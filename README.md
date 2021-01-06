[![Build Status](https://travis-ci.com/cheshir/logrustash.svg?branch=master)](https://travis-ci.com/cheshir/logrustash)
[![Go Report Card](https://goreportcard.com/badge/github.com/cheshir/logrustash)](https://goreportcard.com/report/github.com/cheshir/logrustash)
[![codecov](https://codecov.io/gh/cheshir/logrustash/branch/master/graph/badge.svg)](https://codecov.io/gh/cheshir/logrustash)
[![GoDoc](https://godoc.org/github.com/cheshir/logrustash?status.svg)](https://godoc.org/github.com/cheshir/logrustash)

# Logstash hook for logrus <img src="http://i.imgur.com/hTeVwmJ.png" width="40" height="40" alt=":walrus:" class="emoji" title=":walrus:" />
Use this hook to send the logs to [Logstash](https://www.elastic.co/products/logstash) over both UDP and TCP.

# Important notes

This a patched version of *old* [github.com/bshuster-repo/logrus-logstash-hook](https://github.com/bshuster-repo/logrus-logstash-hook.git) repo.

Added features:

* [Async mode](#async-mode). You can send log messages without blocking logic.
* [Reconnect](#reconnect).

*!* Currently there is no sense to use it. You can get the same behaviour with newest version of package and "smart" TCP connection that will provide reconnects and async behaviour.

## Usage

```go
package main

import (
    "github.com/sirupsen/logrus"
    "github.com/ylamothe/logrustash"
)

func main() {
        log := logrus.New()
        hook, err := logrustash.NewHook("tcp", "172.17.0.2:9999", "myappName")

        if err != nil {
                log.Fatal(err)
        }
        log.Hooks.Add(hook)
        ctx := log.WithFields(logrus.Fields{
          "method": "main",
        })
        ...
        ctx.Info("Hello World!")
}
```

This is how it will look like:

```ruby
{
    "@timestamp" => "2016-02-29T16:57:23.000Z",
      "@version" => "1",
         "level" => "info",
       "message" => "Hello World!",
        "method" => "main",
          "host" => "172.17.0.1",
          "port" => 45199,
          "type" => "myappName"
}
```


## Async mode

Create hook with _NewAsync..._ factory methods if you want to send logs in async mode.

Example:

```go
log := logrus.New()
hook, err := logrustash.NewAsyncHook("tcp", "172.17.0.2:9999", "myappName")
if err != nil {
        log.Fatal(err)
}
log.Hooks.Add(hook)
```

In the very rare cases buffer can be clogged. By default all new messages will be dropped until buffer frees.

If you don't want to lose messages you can change this behaviour:

```go
log := logrus.New()
hook, err := logrustash.NewAsyncHook("tcp", "172.17.0.2:9999", "myappName")
if err != nil {
        log.Fatal(err)
}

hook.WaitUntilBufferFrees = true
log.Hooks.Add(hook)
```

## Reconnect

Doesn't work if you create hook with your own connection. Don't use this factory methods if you want to have auto reconnect:

* NewHookWithFieldsAndConn
* NewAsyncHookWithFieldsAndConn
* NewHookWithFieldsAndConnAndPrefix
* NewAsyncHookWithFieldsAndConnAndPrefix

When occurs not temporary net error hook will automatically try to create new connection to logstash.

With each new consecutive attempt to reconnect, delay before next reconnect will grow up by formula:

`ReconnectBaseDelay * ReconnectDelayMultiplier^reconnectRetries`

Be careful using reconnects without async mode because delay can increase significantly and this will blocks your logic.

Example for async mode:
```go
hook, err := logrustash.NewAsyncHook("tcp", "172.17.0.2:9999", "myappName")
if err != nil {
        log.Fatal(err)
}

hook.ReconnectBaseDelay = time.Second // Wait for one second before first reconnect.
hook.ReconnectDelayMultiplier = 2
hook.MaxReconnectRetries = 10

log.Hooks.Add(hook)
```

With this configuration hook will wait 1024 (2^10) seconds before last reconnect.
When message buffer will full all new messages will be dropped (depends on `WaitUntilBufferFrees` parameter).

Example for sync mode:
```go
hook, err := logrustash.NewHook("tcp", "172.17.0.2:9999", "myappName")
if err != nil {
        log.Fatal(err)
}

hook.ReconnectBaseDelay = time.Second // Wait for one second before first reconnect.
hook.ReconnectDelayMultiplier = 1
hook.MaxReconnectRetries = 3

log.Hooks.Add(hook)
```

WIth this configuration we will have constant reconnect delay in 1 second.

## Hook Fields
Fields can be added to the hook, which will always be in the log context.
This can be done when creating the hook:

```go
hook, err := logrustash.NewHookWithFields("tcp", "172.17.0.2:9999", "myappName", logrus.Fields{
        "hostname":    os.Hostname(),
        "serviceName": "myServiceName",
})
```

Or afterwards:

```go

hook.WithFields(logrus.Fields{
        "hostname":    os.Hostname(),
        "serviceName": "myServiceName",
})
```
This allows you to set up the hook so logging is available immediately, and add important fields as they become available.

Single fields can be added/updated using 'WithField':

```go

hook.WithField("status", "running")
```



## Field prefix

The hook allows you to send logging to logstash and also retain the default std output in text format.
However to keep this console output readable some fields might need to be omitted from the default non-hooked log output.
Each hook can be configured with a prefix used to identify fields which are only to be logged to the logstash connection.
For example if you don't want to see the hostname and serviceName on each log line in the console output you can add a prefix:

```go


hook, err := logrustash.NewHookWithFields("tcp", "172.17.0.2:9999", "myappName", logrus.Fields{
        "_hostname":    os.Hostname(),
        "_serviceName": "myServiceName",
})
...
hook.WithPrefix("_")
```

There are also constructors available which allow you to specify the prefix from the start.
The std-out will not have the '\_hostname' and '\_servicename' fields, and the logstash output will, but the prefix will be dropped from the name.


# TODO

* Add more tests.

# Authors

Name              | Github    | Twitter    |
----------------- | --------- | ---------- |
Boaz Shuster      | ripcurld0 | @ripcurld0 |
Alexander Borisov | cheshir   | cheshirysh |
Klemen Bratec     | klemenb   |            |

# License

MIT.
