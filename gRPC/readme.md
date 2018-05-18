# `gRPC` vs `ASP.NET Core`

## Get started

> In gRPC a client application can directly call methods on a server application on a different machine as if it was a local object, making it easier for you to create distributed applications and services. As in many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, the client has a stub (referred to as just a client in some languages) that provides the same methods as the server.
>
> <https://grpc.io/docs/guides/>

To create simple server/client we need to define contract:

```proto
syntax = "proto3";
option csharp_namespace = "Common.Proto";

message HelloRequest {
    string Name = 1;
}

message HelloResponse {
    string Greeting = 1;
}

service HelloService {
    rpc Greeting(HelloRequest) returns (HelloResponse);
}
```

It allows us to generate strong-typed service and client in any language.

For example I've taken real service definition: <https://github.com/LykkeCity/Lykke.Common.ExchangeAdapter/blob/master/protos/Spot.proto> and used simplest nuget package for code-generation called `Grpc.Tools.MsBuild.Unofficial` - it has some caveats, but it's not so important for start.

Let's reference Nuget package called `Lykke.Common.ExchangeAdapter.Grpc` that contains proto-contract and then implement single method of the defined service:

```c#
public sealed class SpotService : Spot.SpotBase
{
    public override Task<WalletsList> GetWallets(Void request, ServerCallContext context)
    {
        var list = new WalletsList();

        list.Wallets.Add(new Wallet { Asset = "BTC", Balance = 12M, Reserved = 1M });

        return Task.FromResult(list);
    }
}
```

Interesting things to notice - that response `WalletsList` already has the collection `Wallets` defined, it couldn't be null, and all methods have `Task<T>` semantic. Again, we haven't defined any C# DTO manually, all records and enums were generated for us by proto compiler (`protoc`), all we've done - defining messages and services using `protobuf3` syntax. That means that we have the single source of the truth.

Let's run this service and try to query it.

```c#
// server code
private const int Port = 5001;

static async Task Main(string[] args)
{
    var server = new Server
    {
        Services = { Spot.BindService(new SpotService()) },
        Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
    };

    server.Start();

    Console.Write($"Listening port {Port}... Press ENTER to stop the server");
    Console.ReadLine();
    Console.Write("Stopping the server...");

    await server.ShutdownAsync();

    Console.WriteLine(" STOPPED");
}
```

```c#
// client code

private const int Port = 5001;

static async Task Main(string[] args)
{
    var channel = new Channel("localhost", Port, ChannelCredentials.Insecure);
    var client = new Spot.SpotClient(channel);
    var response = await client.GetWalletsAsync(new Void());

    Console.WriteLine(response.ToString());
}
```

```text
$ dotnet Grpc.Demo.Client.dll
{ "wallets": [ { "asset": "BTC", "balance": { "lo": 12 }, "reserved": { "lo": 1 } } ] }
```

> As you can see - we have minimum boilerplate, very clear and concise code, no single line of C# code related to defining contracts, serialization etc. Imagine the effort you need to implement the same using `ASP.NET Core`.

## proto3 language

Comprehensive documentation and language specification available on official site: <https://developers.google.com/protocol-buffers/docs/proto3>

Few things:

1. It doesn't define generalization (generics) on inheritance of messages
1. It doesn't define a lot of standard C# classes, for example it doesn't define `System.Decimal` or `System.DateTime`
1. It has `oneOf` semantic so uncommon for languages like `C#` or `javascript`

### Inheritance and generics

You don't actually need it for defining services, `oneOf` semantic is more powerful (see below).

### Common C# types

`System.DateTime` or `System.Decimal` could be represented as structures of pre-defined types and converted using `implicit` convertion feature of C#. `proto3` has `import` operator so we can define that structures once and then reuse it:

```proto
// Common.proto

syntax = "proto3";

option csharp_namespace = "Lykke.Common.Proto";

message DateTime {
    uint64 ticks = 1;
}

message Decimal {
    int32 lo = 1;          // the first 32 bits of the underlying value
    int32 mid = 2;         // the second 32 bits of the underlying value
    int32 hi = 3;          // the last 32 bis of the underlying value
    int32 signScale = 4;   // the number of decimal digits, and the sign
}

message Void {}
```

```proto
// Spot.proto

syntax = "proto3";

import "Common.proto";

package Lykke.ExchangeAdapter.Spot;

message Wallet {
    string asset = 1;
    Decimal balance = 2;
    Decimal reserved = 3;
}

```

```c#
namespace Lykke.Common.Proto {
    public partial class DateTime
    {
        public static implicit operator System.DateTime(DateTime dateTime)
        {
            return System.DateTime.SpecifyKind(new System.DateTime((long)dateTime.Ticks), System.DateTimeKind.Utc);
        }

        public static implicit operator DateTime(System.DateTime dateTime)
        {
            return new DateTime { Ticks = (ulong)dateTime.Ticks };
        }
    }

    public partial class Decimal
    {
        public static implicit operator System.Decimal(Decimal dec)
        {
            return new System.Decimal(new [] { dec.Lo, dec.Mid, dec.Hi, dec.SignScale });
        }

        public static implicit operator Decimal(System.Decimal dec)
        {
            var bits = System.Decimal.GetBits(dec);

            return new Decimal
            {
                Lo = bits[0],
                Mid = bits[1],
                Hi = bits[2],
                SignScale = bits[3]
            };
        }
    }
}
```

### oneOf semantic

`oneOf` refers to absent feature of Algebraic Data Type in C# language. C# doesn't have [sum-types](https://en.wikipedia.org/wiki/Tagged_union), also called a tagged-union, variant, variant record, choice type, discriminated union, disjoint union. For contrast - F# defines such types, they called [Discriminated Unions](https://fsharpforfunandprofit.com/posts/discriminated-unions/).

For us the importance of such types is possibility to define negative scenarios (errors) in contract. Proto3 (as many languages) doesn't have exceptions so we have to define errors explicitly.

```proto
enum CreateLimitOrderError {
    VolumeTooSmall = 0;
    NotEnoughBalance = 1;
    IncorrectPrice = 2;
    IncorrectTradeType = 3;
    IncorrectInstrument = 4;
}

message CreateLimitOrderResult {
    oneOf result {
        string OrderId = 1;
        CreateLimitOrderError error = 2;
    }
}
```

And as a result we have to check if we've got positive response or not. For reference types we can't just check for nullness of the respond, however it's impossible for value-types, so we have to check special "discriminator" field of our message instance. Hopefully, `protoc` defined it for us:

```c#
if (response.ResultCase == CreateLimitOrderResult.ResultOneofCase.OrderId)
{
    Console.WriteLine($"Created order: {response.OrderId}");
}
else
{
    Console.WriteLine($"An error occured: {response.Error:G}");
}
```

It might seem that the code tends to be more verbose, but it solves several issues:

1. It distinguish business and validation errors from transport and infrastructure exceptions.
1. It explicitly defines all errors in the contract.
1. It forces a developer to check result of the call.

> For `ASP.NET Core` we have implemented serialization of standard .NET types (thanks to `Newtonsoft.Json`), although we need to implement a middleware that intercepts exceptions, map them to HttpStatusCodes, serialize error to response content, implement `DelegatingHandler` on the client side, that handles non-success status codes, reads content of the error, deserializes it to corresponding exceptions and then throws them. And it likely wont be a generic code for the whole service, it might vary for each particular REST-method.

## Authentication / Headers / Intercepters

> gRPC is designed to work with a variety of authentication mechanisms, making it easy to safely use gRPC to talk to other systems. You can use our supported mechanisms - SSL/TLS with or without Google token-based authentication - or you can plug in your own authentication system by extending our provided code.
>
> <https://grpc.io/docs/guides/auth.html>

For C# there are only few methods out of the box (Client Certificates and Google-based tokens).

Adding custom authentication using custom headers could be made in the same way as for `ASP.NET Core`. Each call has corresponding `CallContext`/`ServerContext` and we can intercept the calls change those contexts programmatically.

```c#
public class XApiAuthInterceptor : Interceptor
{
    public const string XAPI = "X-API-KEY";

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var h = context.Options.Headers.FirstOrDefault(x => x.Key == XAPI);

        if (h == null)
        {
            return new AsyncUnaryCall<TResponse>(
                Task.FromResult(default(TResponse)),
                Task.FromResult(context.Options.Headers),
                () => new Status(StatusCode.Unauthenticated, ""),
                () => context.Options.Headers,
                () => { });
        }
        else
        {
            return base.AsyncUnaryCall(request, context, continuation);
        }
    }
}

// usage
var spotControllerImpl = new SpotControllerImpl();

var serverServiceDefinition = Spot
    .BindService(spotControllerImpl)
    .Intercept(new XApiAuthInterceptor());
```

> Unfortunately, there is no convenient way to mark some methods with some kind of attributes or operation filters as it could be done using `ASP.NET Core`. Afaik there is system of plugins those allow to extend `protoc` functionality but it doesn't seem "easy-to-go" thing.

## Validation

`gRPC` doesn't define any validation features, the most convenient way to implement validation - using interceptors. All DTO's defined as partial classes so we could extend them implementing some kind of interface `IValidate`. It might not be as cool as possibility to mark fields with `[Required]` attribute but it works, I think it's possible to use a some validation framework.

## Performance

In terms of performance we could measure various metrics - size of messages, latency and throughput for dumb services.

### Message sizes

### Latency

### Throughput

### Recovery

### Monitoring

## Toolchain

### Web UI for testers

### REST over gRPC

## .NET Core code generation and integration