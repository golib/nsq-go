nsq-go [![CircleCI](https://circleci.com/gh/golib/nsq-go.svg?style=shield)](https://circleci.com/gh/golib/nsq-go) [![Go Report Card](https://goreportcard.com/badge/github.com/golib/nsq-go)](https://goreportcard.com/report/github.com/golib/nsq-go) [![GoDoc](https://godoc.org/github.com/golib/nsq-go?status.svg)](https://godoc.org/github.com/golib/nsq-go)
======

Go package providing tools for building NSQ clients, servers and middleware.

Motivations
-----------

We ran into production issues with the standard nsq-go package where our workers
would enter deadlock situations where they would stop consuming messages and
just hang in endless termination loops.  
After digging through the code and trying to figure out how to address the
problem it became clear that a rewrite was going to be faster than dealing with
the amount of state synchronization that was done in nsq-go (multiple mutexes,
channels and atomic variables involved).

This package is designed to offer less features than the standard nsq-go package
and instead focus on simplicity and ease of maintenance and integration.

Consumer
--------

```go
package main

import (
    "github.com/golib/nsq-go"
)

func main() {
    // Create a new consumer, looking up nsqd nodes from the listed nsqlookup
    // addresses, pulling messages from the 'world' channel of the 'hello' topic
    // with a maximum of 250 in-flight messages.
    consumer, _ := nsq.StartConsumer(nsq.ConsumerConfig{
        Topic:   "hello",
        Channel: "world",
        Lookup:  []string{
            "nsqlookup-001.service.local:4161",
            "nsqlookup-002.service.local:4161",
            "nsqlookup-003.service.local:4161",
        },
        MaxInFlight: 250,
    })

    // Consume messages, the consumer automatically connects to the nsqd nodes
    // it discovers and handles reconnections if something goes wrong.
    for msg := range consumer.Messages() {
        // handle the message, then call msg.Finish or msg.Requeue
        // ...
        msg.Finish()
    }
}
```

Producer
--------

```go
package main

import (
    "github.com/golib/nsq-go"
)

func main() {
     // Starts a new producer that publishes to the TCP endpoint of a nsqd node.
     // The producer automatically handles connections in the background.
    producer, _ := nsq.StartProducer(nsq.ProducerConfig{
        Topic:   "hello",
        Address: "localhost:4150",
    })

    // Publishes a message to the topic that this producer is configured for,
    // the method returns when the operation completes, potentially returning an
    // error if something went wrong.
    producer.Publish([]byte("Hello World!"))

    // Stops the producer, all in-flight requests will be canceled and no more
    // messages can be published through this producer.
    producer.Stop()
}
```

segment/nsq
-----------

The [segment/nsq](https://hub.docker.com/r/segment/nsq/tags) docker image packages
the tools within this repository:
- **nsq-to-nsq** is similar to the standard nsq_to_nsq tool but supports rate limiting.
- **nsqlookup-proxy** is a proxy for nsqlookupd, it aggregates the results from querying
multiple nsqlookupd servers and exposes them under a single endpoint.
