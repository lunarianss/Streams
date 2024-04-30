# Streams

![图片展示](icon/stream.svg) Streams is a [Grpc Go](https://github.com/grpc/grpc-go) Proxy server

## Project Goal

Build a transparent reverse proxy for gRPC targets that will make it easy to expose gRPC services
over the internet. This includes:

- no needed knowledge of the semantics of requests exchanged in the call (independent rollouts)
- easy, declarative definition of backends and their mappings to frontends

The project now exists as a **proof of concept**, with the key piece being the `proxy` package that
is a generic gRPC reverse proxy handler.

## Representations

The original project is from [mwitkow/grpc-proxy](https://github.com/mwitkow/grpc-proxy), mwitkow was a great programmer and i admired him greatly. streams for the original project, adapted to the latest grpc standard and the latest golang version, and for the project's core functions, made a comment description. In addition, all the tests of the project have been passed. Streams project is also used as a dependency package of my gateway project [Forest](https://github.com/Ryan-eng-del/Forest) to provide Grpc proxy.

## Proxy Handler

The package [`proxy`](proxy/) contains a generic gRPC reverse proxy handler that allows a gRPC server to
not know about registered handlers or their data types. Please consult the docs, here's an exaple usage.

Defining a `StreamDirector` that decides where (if at all) to send the request

```go
director := func(ctx context.Context, fullMethodName string) (context.Context, *grpc.ClientConn, error) {
    // Make sure we never forward internal services.
    if strings.HasPrefix(fullMethodName, "/com.example.internal.") {
        return ctx, nil, grpc.Errorf(codes.Unimplemented, "Unknown method")
    }
    md, ok := metadata.FromIncomingContext(ctx)
    if ok {
        // Decide on which backend to dial
        if val, exists := md[":authority"]; exists && val[0] == "staging.api.example.com" {
            // Make sure we use DialContext so the dialing can be cancelled/time out together with the context.
            c, err := grpc.DialContext(ctx, "api-service.staging.svc.local", grpc.WithCodec(proxy.Codec()),grpc.WithInsecure())
            return ctx, c, err
        } else if val, exists := md[":authority"]; exists && val[0] == "api.example.com" {
            c, err := grpc.DialContext(ctx, "localhost:50055", grpc.WithCodec(proxy.Codec()),grpc.WithInsecure())
            return ctx, c, err
        }
    }
    return ctx, nil, grpc.Errorf(codes.Unimplemented, "Unknown method")
}
```

Then you need to register it with a `grpc.Server`. The server may have other handlers that will be served
locally:

```go
server := grpc.NewServer(
    grpc.CustomCodec(proxy.Codec()),
    grpc.UnknownServiceHandler(proxy.TransparentHandler(director)))
pb_test.RegisterTestServiceServer(server, &testImpl{})
```

## License

`grpc-proxy` is released under the Apache 2.0 license. See [LICENSE.txt](LICENSE.txt).
