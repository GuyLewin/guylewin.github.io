---
title: 'Inconsistencies in grpc-java Error Handling with Bidirectional Streams'
date: '2020-10-27T02:37:59+00:00'
author: Guy Lewin
layout: post
redirect_from:
  - /inconsistencies-in-grpc-java-error-handling-with-bidirectional-streams/
tags:
  - bidirectional
  - grpc
  - grpc-java
  - oncompleted
  - onerror
  - statusruntimeexception
  - stream
  - streamobserver
---

While working on a [grpc-java](https://github.com/grpc/grpc-java) project with [bidirectional streaming](https://grpc.io/docs/languages/java/basics/#bidirectional-streaming-rpc) I noticed lack of documentation on how to handle errors. I wanted to know when are errors thrown, and how should an error be handled after receiving one.

Since I could barely find any documentation online, I constructed a few tests of my own. I arranged a small project with a bidirectional gRPC service, and configured [JUnit](https://junit.org/) to perform a successful handshake over the loopback between a client and a server before each test. I was using the most recent version of `grpc-java` (while writing this post) - `1.33.0`. There are 4 `StreamObserver` objects in all the tests:

- Client Request - Implemented by `grpc-java`, calling its `onNext()`, `onError()` and `onCompleted()` should trigger the appropriate Server Request `StreamObserver` object over the network.
- Client Response - Implemented by me, receiving triggers from Server Response `StreamObserver` object over the network.
- Server Request - Implemented by me, receiving triggers from Client Response `StreamObserver` object over the network.
- Server Response - Implemented by `grpc-java`, calling its `onNext()`, `onError()` and `onCompleted()` should trigger the appropriate Client Response `StreamObserver` object over the network.

In the first group of tests - I wanted to check what `onError()` callbacks are triggered on the `StreamObserver` objects when I call the 2 different `onError()` `grpc-java` implementations. The columns represent the `StreamObserver` the error was sent on (using `onError()`). The rows represent the StreamObserver error was checked on (also, using `onError()`):

|  | **Client Request** | **Server Response** |
|---|---|---|
| **Server Request** | `StatusRuntimeException("CANCELLED: client cancelled")`, `cause` is `null`. | `onError()` not triggered |
| **Client Response** | `StatusRuntimeException("CANCELLED: Cancelled by client with StreamObserver.onError()")`, original exception included as `cause`. | `StatusRuntimeException("UNKNOWN")`, `cause` is `null`. |

(Columns represent the StreamObserver the error was sent on. Rows represent the StreamObserver error was checked on)

As you can see - the results are very confusing. Each scenario behaved differently, especially in Server Response `StreamObserver` object where `onError()` wasn’t even called when the error was sent from the Server Request object. This proves it is wrong to rely on `onError()` always being called on both sides when an error is sent.

The test above showed us when and how `onError()` is being triggered on listening `StreamObserver` objects. But what can (and should) you do with a such object after receiving an error? Should you call `onCompleted()` manually? Should you call `onError()` on the corresponding side of the `StreamObserver`?

According to [grpc-java’s StreamObserver documentation](https://grpc.github.io/grpc-java/javadoc/io/grpc/stub/StreamObserver.html), `onError()` and `onCompleted()` should only be called once and should be the last methods called on an instance. But does that apply if it was called by gRPC over the network? I performed some tests by calling `onNext()` and `onComplete()` after throwing errors. These are the results:

|  | **Client Request** | **Server Response** |
|---|---|---|
| **Client Request - onNext** | `IllegalStateException` | No exception |
| **Client Request - onCompleted** | `IllegalStateException` | No exception |
| **Server Response - onNext** | `StatusRuntimeException` | `IllegalStateException` |
| **Server Response - onCompleted** | No exception | `IllegalStateException` |

(Columns represent the StreamObserver the error was sent on. Rows represent the message sent after the error)

Once again there are inconsistencies in how gRPC notifies us on the error. It seems like it’s wrong to use the stream after an error was thrown in any way, but only in some cases an exception is thrown back to the caller of `onNext()` or `onCompleted()`. I was bothered to see that calling `onNext()` and `onCompleted()` on a Server Response `StreamObserver` object after receiving an error from Client Request side didn’t result in the same exception.

In conclusion, based on the tests I performed it appears that:

- Sending errors from the client to the server will always call `onError()` on all `StreamObserver` objects, with informative errors. The other way around isn’t as robust.
- Streams shouldn’t be used after an error was received, not even to call `onCompleted()`. gRPC sometimes throws exceptions when calling methods on closed streams.

The GitHub repository with all of my tests can be found [here](https://github.com/GuyLewin/grpc-bidirectional-streaming-error-handling).