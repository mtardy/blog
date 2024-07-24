---
title: "gRPC-Go: Built-in Client Retry Mechanism"
date: "2024-07-24"
tags: ["Go", "open-source", "gRPC"]
categories: ["gRPC"]
slug: "grpc-go-client-retry"
description: "A little guide to use gRPC-Go built-in client retry mechanism."
---

gRPC recently deprecated its `grpc.Dial` in favor of the new `grpc.NewClient`.
The latter performs no I/O and lazily connects when you are eventually doing a
RPC call. It changed the way we perform retry in `tetra`, the CLI of
[Tetragon](https://github.com/cilium/tetragon) because we previously used the
connect that happened at the client creation (with `grpc.Dial` and `WithBlock`)
to make sure the connection was possible, and retry in this case. This wasn't
extremely resilient but had the good taste of being in a single place for all
subcommands that will make an RPC call: during client creation.

So given the ["anti-patterns" documentation](https://github.com/grpc/grpc-go/blob/master/Documentation/anti-patterns.md)
that was written to justify the deprecation, what we were doing was "Especially
bad"... What piqued my interest is that the document is a bit contradictory, it
says in the best practices:
> When retrying failed RPCs, consider using the built-in retry mechanism
> provided by gRPC-Go, if available, instead of manually implementing retries.
> Refer to the [gRPC-Go retry example
> documentation](https://github.com/grpc/grpc-go/blob/master/examples/features/retry/README.md)
> for more information. Note that this is not a substitute for client-side
> retries as errors that occur after an RPC starts on a server cannot be
> retried through gRPC's built-in mechanism.

But just a few lines later, they "manually implement retries", in the same
fashion as what is done built-in in the library.

## Manually implementing retry

Back to my situation, after reimplementing our gRPC client creations with
`grpc.NewClient` instead of blocking on `grpc.Dial`, I was left without any
retry mechanisms, and for some reason, we had a built-in retry option in
`tetra`. So first I tried to do what the "anti-patterns" post did, not what
it said, and reimplement my own retry mechanism.

```Go
// ExpBackoff retries calling the function f as many time as specified by the
// global var Retries retrieved from flag. You must pass the context of the
// calling client to also respect interrupt and timeout.
func ExpBackoff[T any](ctx context.Context, f func() (T, error)) (res T, err error) {
	backoff := time.Second
	for attempt := range Retries {
		res, err = f()
		if err != nil {
			if status.Code(err) != codes.Unavailable {
				break
			}

			logger.GetLogger().WithFields(logrus.Fields{
				"attempt": attempt + 1,
				"backoff": backoff,
			}).WithError(err).Error("RPC call failed, retrying...")

			select {
			case <-time.After(backoff):
			case <-ctx.Done():
				break
			}

			backoff *= 2
			continue
		}
	}
	return
}
```

The good part is that I could finally write some Go code using the "new"
generics which made me happy but the downside of this is that:
1. It's broken code if retry equals zero, but that's just a detail!
2. You need to change all of your RPC calls to look like this:

```diff
- res, err = c.Client.AddTracingPolicy(c.Ctx, &mypkg.MyRPCRequest{})
+ res, err = common.ExpBackoff(c.Ctx, func() (*mypkg.MyRPCResponse, error) {
+     return client.MyRPC(c.Ctx, &tetragon.MyRPCRequest{})
+ })
```

## gRPC-Go client retry

I wanted to give it a try to their built-in retry mechanism so I could remove
my buggy implementation and it would be transparently doing retry without even
touching the code.

### History of this feature

gRPC has a Request For Comments process, the gRFCs (ah!), and [RFC
A6](https://github.com/grpc/proposal/blob/master/A6-client-retries.md) is about
client retry design. Those RFCs are designed to be language agnostics and are
eventually implemented in most of the languages gRPC supports.

The Go implementation was added to gRPC-Go in June 2018 by PR #2111 "[client:
Implement gRFC A6: configurable client-side retry support](https://github.com/grpc/grpc-go/pull/2111)".
Unfortunately, it [suffered from a hardcoded `maxAttempts` to 5](https://github.com/grpc/grpc-go/issues/4615)
which made the whole implementation a bit limited. Fortunately, it was fixed
and merged this May with PR #7229 "[client: implement maxAttempts for retryPolicy](https://github.com/grpc/grpc-go/pull/7229)",
so part of the `v1.65.0` release tag that I was supposed to upgrade to with the
deprecation handling.

### How to enable it

There is [an example](https://github.com/grpc/grpc-go/blob/v1.65.0/examples/features/retry/README.md)
that shows how to enable and configure retry, but this was not enough
documentation for me, I had to search the code to understand how this worked,
so here is my guide.

To enable client-side retry, you need a few things:
1. Find your service name, it's the protobuf package dot the service name that
   groups the RPC functions. So in my case, it was
   `tetragon.FineGuidanceSensors`.
2. Configure the retry policy via a method config *in JSON*. Indeed, it might
   looks weird that you cannot use a proper Golang struct as a param but it's
   not proposed yet, the [`type RetryPolicy struct`](https://github.com/grpc/grpc-go/blob/v1.65.0/internal/serviceconfig/serviceconfig.go#L157)
   is still in `internal` in the grpc-go repository.
3. Make sure you also pass a `grpc.WithMaxCallAttempts` DialOption to bump the
   `maxAttempts` above 5. Indeed, even you will find a `maxAttempts` in the
   `retryPolicy`, there is another `maxAttempts` (the one that was hardcoded)
   that will cap the maximum of attempts later on. See more [in the
   code](https://github.com/grpc/grpc-go/blob/v1.65.0/service_config.go#L284-L286).

In the end, it might look similar to this:

```Go
grpc.NewClient(address,
	grpc.WithDefaultServiceConfig(`{
		"methodConfig": [{
			"name": [{"service": "<package>.<service>"}],
			"retryPolicy": {
				"MaxAttempts": 10,
				"InitialBackoff": "1s",
				"MaxBackoff": "60s",
				"BackoffMultiplier": 2,
				"RetryableStatusCodes": [ "UNAVAILABLE" ]
		}
	}]}`),
	grpc.WithMaxCallAttempts(10), // maxAttempt includes the first call
)
```

You see you can customize the various backoff parameters. When doing
experimentation, that might not be very clear so note that the final backoff
duration is completely random and chosen between 0 and the final duration
computed via the given parameters. See more [in the code](https://github.com/grpc/grpc-go/blob/v1.65.0/stream.go#L702).

Again, this will be transparent, so to the Go RPC user, you will just see your
RPC call hangs for a longer time when it's retrying. To see it in action without
touching the code, make the gRPC server unreachable and run your code with the
`GRPC_GO_LOG_SEVERITY_LEVEL` env variable set to `warning`.

Note that logs don't always have the time to be pushed before exit so the
output might be a bit off, but the number of retries is respected. To verify
that, you can debug or synchronously print in the [`stream.go:shouldRetry`](https://github.com/grpc/grpc-go/blob/v1.65.0/stream.go#L622)
or [`stream.go:withRetry`](https://github.com/grpc/grpc-go/blob/v1.65.0/stream.go#L753).

