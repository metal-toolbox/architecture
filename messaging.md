---
title: Event stream
description: Event stream and messaging properties
tags:
  - events
author: joel.rebello@eu.equinix.com
status: proposed
---

### Table of contents

1. [Introduction](#introduction)
2. [Terminology](#terminology)
    1. [Streams](#streams)
    2. [Consumers](#consumers)
    3. [Subscribers](#subscribers)
    4. [Subjects](#subjects)
    5. [Controllers](#controllers)
    6. [Orchestrators](#orchestrators)
3. [Events library](#events-library)
4. [Configuration](#configuration)
    1. [Stream](#stream)
    2. [Consumer](#consumer)
    3. [Subjects](#subjects)
5. [Event message format](#event-message-format)


## Introduction

The services part of the metal-toolbox project communicate with each other to take actions on the hardware they run inventory,
firmware installs and various other actions on.

This interaction between services is described in the [architecture doc](firmware-install-service.md#architecture).

The metal-toolbox services receive and transmit events to perform work, this manner of communication enables them to be decoupled and asynchronous.

While the services strive to be asynchronous, in some cases the services call out to APIs in a synchronous manner, in such cases the interaction is either transactional or the service is at a decision point in its work.

This document covers the following points,

 - The required event stream properties for the metal-toolbox use case.
 - The messages sent on the event stream.
 - The event stream message format.

The event stream technology picked for this purpose is the [NATS Jetstream](https://docs.nats.io/nats-concepts/jetstream).

### Terminology

#### Streams

A stream is a NATS Jetstream setup either by an administrator or by the application itself at startup, the stream is where multiple `subscribers` connect as `consumers` and can transmit, receive messages on the configured subjects. Messages sent on the stream can be persisted based on its configuration.

In the metal-toolbox context, a service will attempt to setup the stream if one doesn't already exist.

#### Consumers

A `consumer` is a NATS Jetstream view created on an existing stream - either by an administrator or application, the `consumer` view may or may not be limited to one or more subjects.

Subscribers connect as a NATS Jetstream `consumer`. Consumers can be configured to be push or pull based and can be configured to require explicit acknowledgements for events received.

In the metal-toolbox context, a service will attempt to setup the consumer if one doesn't already exist.

#### Subscribers

Subscribers connects to a stream as a `consumer`, multiple subscribers can connect as a `consumer` and form groups.

Subscribers are `ephermal` if they don't specify a durable name. In the metal-toolbox setup, all subscribers are durable.

#### Subjects

Subjects are strings configured to scope events to publishers and subscribers on a NATS Jetstream,
more about this can be read here [Subject Based Messaging](https://docs.nats.io/nats-concepts/subjects)

#### Controllers

Each controller subscribed to the stream acks/nacks received events based on its configuration - if its capable of fulfilling the Condition included in an event.

see [architecture doc](firmware-install-service.md#controllers)

#### Orchestrators

see [architecture doc](firmware-install-service.md#condition-orchestrator---conditionorc)


## Event stream properties

These were the properties of the event stream that are required
by the metal-toolbox services,

 1. Persistent queue.
 2. Workload distribution.
 3. Stateless workers.
 4. Only once delivery.
 4. Explicit acknowledgements.
 5. Rate limiting.

The next section describes how the NATS Jetstream properties full fill the requirements mentioned above.

#### Persistent queue

Events should not be lost if a subscriber is not currently
on the stream, events sent on the NATS Jetstream are persisted until they are consumed and `ack`'ed by a subscriber.

In the metal-toolbox context the stream is configured as a `WorkQueue` and to require acknowledgements from subscribers.

#### Workload distribution

Multiple subscribers can form a `Queue Group` while subscribing to a common `consumer` subject. This enables horizontal scalability of subscribers that perform work, since workers can be added based on demand.

In the metal-toolbox context the event stream is configured as a `WorkQueue`. And the `consumer` is setup to be `pull` based

#### Once and only once delivery

- Each event is delivered only once to ensure theres no duplicate work being performed.

In the metal-toolbox context events wont't be published if there are no subscribers, this is because the stream is configured as a `WorkQueue`,
and `WorkQueue` streams expects one or more consumers to be available for publishing an event.

#### Explicit acknowledgements

The `consumer` is configured to require an acknowledgements within a time window, if an acknowledgement is not received for an event, the event will be republished.

A subscriber acknowledges the event with an `Ack` - when its either completed or failed. While the work is in progress the subscriber sends an `InProgress()` ack, it must do this within the acknowledgement time window - `Ack Wait` configured on the consumer.

If a subscriber figures it cannot proceed with the work itself and it needs to be retried by another subscriber, it `Nack`s the event.

## Events library

The `controllers` and `orchestrator` and other services participating in the event stream
make use of the shared [hollow-toolbox/events](https://github.com/metal-toolbox/hollow-toolbox/tree/events/events) library. This library ensures clients connect to the stream follow a common pattern for configuration and it provides an abstraction over the event stream for flexibility in testing.

When using this library services can serialize and deserialize events sent in a common format,
the format of messages is described in the [event message format](#event-message-format).

## Configuration

In th metal-toolbox context, the stream and consumer configurations are setup
by the `controllers` or `orchestrators` that connect as `consumer`s to the stream.

The stream and consumers are initialized only if the expected stream does not already exist. The `controllers` and `orchestrators` must be provided stream credentials that enables them to initialize the stream.

#### Stream

This is the stream configuration as setup by a `controller` when
connecting to the NATS Jetstream service.

```bash
~ # nats -s nats://serverservice:password@nats:4222 stream info
? Select a Stream controllers
Information for Stream controllers created 2023-03-08 06:32:52

             Subjects: com.hollow.sh.controllers.commands.>, com.hollow.sh.controllers.replies.>
             Replicas: 1
              Storage: File

Options:

            Retention: WorkQueue
     Acknowledgements: true
       Discard Policy: Old
     Duplicate Window: 2m0s
    Allows Msg Delete: true
         Allows Purge: true
       Allows Rollups: false

Limits:

     Maximum Messages: unlimited
  Maximum Per Subject: unlimited
        Maximum Bytes: unlimited
          Maximum Age: unlimited
 Maximum Message Size: unlimited
    Maximum Consumers: unlimited

 ...
```

#### Consumer

This is the stream `consumer` configuration as setup by a `controller` connecting to a stream.

```bash
? Select a Stream controllers
? Select a Consumer controller-alloy
Information for Consumer controllers > controller-alloy created 2023-03-08T06:39:11Z

Configuration:

                Name: controller-alloy
    Delivery Subject: controllers.alloy
      Filter Subject: com.hollow.sh.controllers.commands.>
      Deliver Policy: All
 Deliver Queue Group: controllers
          Ack Policy: Explicit
            Ack Wait: 5m0s
       Replay Policy: Instant
     Max Ack Pending: 10
        Flow Control: false
 ...
```

#### Subjects

The [`Serverservice`](https://github.com/metal-toolbox/hollow-serverservice) server inventory publishes events for new `server` objects created. These events are received by the [`Condition Orchestrator`](https://github.com/metal-toolbox/conditionorc/tree/mvp) and forwarded to controllers.


| service                | role       | subject                                   |
|------------------------|------------|-------------------------------------------|
| Serverservice          | Publisher  |    `com.hollow.sh.serverservice.events.>` |
| Condition Orchestrator | Subscriber |    `com.hollow.sh.serverservice.events.>` |


The `Condition Orchestrator`, `Condition Orchestrator API` forwards work for controllers through events containing [`Conditions`](firmware-install-service.md#conditions) for controllers to reconcile (in the above case - collect out of band inventory).


| service                    | role       | subject                                   |
|----------------------------|------------|-------------------------------------------|
| Condition Orchestrator     | Publisher  | `com.hollow.sh.controllers.commands.>`    |
| Condition Orchestrator API | Publisher  | `com.hollow.sh.controllers.commands.>`    |
| Alloy (Controller)         | Subscriber | `com.hollow.sh.controllers.commands.>`    |
| Flasher (Controller)       | Subscriber | `com.hollow.sh.controllers.commands.>`    |
| Controllers (all)          | Publisher  | `com.hollow.sh.controllers.replies.>`     |


## Event message format

The notes below describe the format of the event messages sent by `subscribers` and `publishers`
on the metal-toolbox event stream.

In most cases services using the events library [hollow-toolbox events](https://github.com/metal-toolbox/hollow-toolbox/tree/events/events) do not have to deal with the message structure directly. The fields are described here for reference purposes only and users should check the library for more current information.

Events sent using the events library are sent in the [pubsubx.Message](https://github.com/infratographer/x/blob/main/pubsubx/message.go) format, which consists of the below fields.


```go
type Message struct {
	// SubjectURN is a string representing the identity of the topic of this message
	SubjectURN string `json:"subject_urn"`
	// EventType describes the type of event that has triggered this message
	EventType string `json:"event_type"`
	// AdditionalSubjectURNs is a group of strings representing additional identities associated with this message
	AdditionalSubjectURNs []string `json:"additional_subjects"`
	// ActorURN is a string representing the identity of the actor that created this message
	ActorURN string `json:"actor_urn"`
	// Source is a string representing the identity of the source system that created the message
	Source string `json:"source"`
	// Timestamp is the time representing when the message was created
	Timestamp time.Time `json:"timestamp"`
	// SubjectFields is a map of additional descriptors for this message
	SubjectFields map[string]string `json:"fields"`
	// AdditionalData is a field to store any addition information that may be important to include with your message
	AdditionalData map[string]interface{} `json:"additional_data"`
}
```

Here we describe some of the important fields,

1. The `SubjectURN` field is typically in the form of `<namespace>:<resourceType>:<resourceID>`

Where,

 - `<namespace>` is set to `hollow`.
 - `<resourceType>` is set to `servers`.
 - `<resourceID>` is the identifier for the resource type..


The events library parses the  *URN fields using the [urnx](https://github.com/infratographer/x/tree/main/urnx) package. A message with an invalid URN is will be dropped and logged by the service receiving it.


2. The `EventType` is upto the producer and consumer of the event, in the `Serverservice` context, this field is set to `Create`, `Update`, `Delete`.
In the context of the Condition orchestrator and controllers this is set to a [`Condition Kind`](https://github.com/metal-toolbox/conditionorc/blob/mvp/pkg/types/types.go#L15).

3. TODO describe the `ActorURN` field when its actually in use, at the moment its not.

4. The `Source` field is set to the service name that transmitted the event.

5. The `Timestamp` field includes the timestamp in UTC for when the event was created for sending.

6. The `AdditionalData` field is where the `Condition` required to full filled is included.
