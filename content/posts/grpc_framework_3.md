---
title: "Part 3: Demystifying gRPC"
date: 2020-01-20
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

gRPC describes itself as:
> gRPC is a modern open source high performance RPC framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services.

This is Part 3 of a series of three articles describing gRPC functionality. It's not necessary to read the first two parts but it will give you a better context if you do. [Part 1](https://dev-state.com/posts/grpc_framework_1/) & [Part 2](https://dev-state.com/posts/grpc_framework_2/)

In this part I'll explain the pluggable functionality of gRPC, more concrete I'll create  a couple of gRPC middlewares which will be applied to both Unary and Streaming calls. gRPC middlewares are named Interceptors.  Interceptors are very useful to wrap functionality around a RPC call. It helps to separate things like logging/auth/monitoring/tracing from the logic of the RPC service.

## gRPC Interceptors

In gRPC there are two kinds of interceptors, **unary** and **stream**. Unary interceptors handle single request/response RPC calls whereas stream interceptors handle RPC calls for messages streams.You don't have to create the interceptors from scratch, unless the desired functionality is not covered by the available library. For Go, there are a number of Interceptors already available and can be plugged in your code, here is the [Link](https://github.com/grpc-ecosystem/go-grpc-middleware). 

For the sake of this exercise I'll create a new Interceptor from scratch, and explain each step of the process. For logging there are already two good interceptors available, [Logrus](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/master/logging/logrus) and [Zap](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/master/logging/zap). I'm going to create a new one for [Klog](https://github.com/kubernetes/klog) which is the logging package used within Kubernetes. I plan to follow as close as possible the structure of the logrus and zap implementation. Klog is a simple logging mechanism, which is a fork of [Glog](https://github.com/golang/glog). [Klogr](https://github.com/kubernetes/klog/tree/master/klogr) implements the [logr interface](https://github.com/go-logr/logr) in terms of Kubernetes' klog. This provides a relatively minimalist API to logging in Go, backed by a well-proven implementation.

[The V0.4](https://github.com/danrusei/grpc_framework/tree/v0.4) tag contain the installed Klog Interceptor.

Although it is possible to create a gRPC Interceptor for other languages, due to slim support and the low level of maturity for Python and others, I'll implement only to GO code. If you are curious about the latest on gRPC Python Interceptor implementation, I would suggest to read [this issue](https://github.com/grpc/grpc/issues/18191) raised on github.

## Unary Interceptor

Unary interceptor intercepts unary RPC calls. An implementation of a unary interceptor can usually be divided into three parts: pre-processing, invoking RPC method, and post-processing.
For pre-processing, users can get info about the current RPC call by examining the args passed in, such as RPC context, method string, request to be sent, and CallOptions configured. With the info, users can even modify the RPC call. After pre-processing is done, user can invoke the RPC call by calling the invoker. Once the invoker returns the reply and error, user can do post-processing of the RPC call. Usually, it's about dealing with the returned reply and error.

### Unary Client Interceptor

UnaryClientInterceptor intercepts the execution of a unary RPC on the client. Invoker is the handler to complete the RPC and it is the responsibility of the interceptor to call it. It is a function type with the signature: 

{{< code language="go" isCollapsed="false" >}}
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
{{< /code >}}

In my implementation, I wrap the the grpc.UnaryClientInterceptor within the UnaryClientInterceptor function to evaluate the options provided by users.

{{< code language="go" isCollapsed="false" >}}
func UnaryClientInterceptor(log logr.Logger, opts ...Option) grpc.UnaryClientInterceptor {
	o := evaluateClientOpt(opts)
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		request := req.(*api.ClientRequestType)
		log.Info("requesting all product types from vendor: " + request.GetVendor())
		fields := newClientLoggerFields(ctx, method)
		startTime := time.Now()
		err := invoker(ctx, method, req, reply, cc, opts...)
		logFinalClientLine(o, log, fields, startTime, err, "finished client unary call")
		return err
	}
}
{{< /code >}}

We use dependency injection to get the **logr.Logger** which is implemented by klog and the **options** that it is a variadic function that accepts a variable number of *grpcklog.Option*, that is explained after the next paragraph.

But first let's have a look over the pre-processing part. We retrieve the **vendor** from the **ClientRequestType** and log it with klog Logger. **newClientLoggerFields** returns a map[string]interface{} that holds a number of values, like the method and service called. **starTime** records the time before the RPC is called by the invoker.
**logFinalClientLine** function takes all the available parameters, convert the invoker error to the canonical error codes used by gRPC, find the logging level, calculate the duration of the invoker call and log these informations. Check out the **Run the Service** part to see the results.

Going back to Option functionality, I'll not cover it in detail due to space constrains, you can find the implementation [over here](https://github.com/danrusei/grpc_framework/blob/v0.4/middleware/grpcklog/options.go). The main idea is that the user can call the Interceptor with a number of options:
 
 * **WithDecider** customizes the function for deciding if the gRPC interceptor logs should log.
 * **WithLevels** customizes the function for mapping gRPC return codes and interceptor log level statements.
 * **WithCodes** customizes the function for mapping errors to error codes.
 * **WithDurationField** customizes the function for mapping request durations to fields.

The implementation is similar to the ones provided by logrus and zap. If the user do not provide any options then the default is applied. **DefaultClientCodeToLevel** is the default implementation of gRPC return codes to **KlogLevel** for client side. KlogLevel is the type which define the INFO, WARNING and ERROR log levels.

{{< code language="go" isCollapsed="false" >}}
func evaluateClientOpt(opts []Option) *options {
	optCopy := &options{}
	*optCopy = *defaultOptions
	optCopy.levelFunc = DefaultClientCodeToLevel
	for _, o := range opts {
		o(optCopy)
	}
	return optCopy
}
{{< /code >}}

### Unary Server Interceptor

Unary server Interceptor has the signature:

{{< code language="go" isCollapsed="false" >}}
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
{{< /code >}}

It provides a hook to intercept the execution of a unary RPC on the server and it allow us to modify the response returned from the gRPC call. Context is used for timeouts but also to add/retrieve request metadata. **info** is the information on the gRPC server that is handling the request. **handler** has to be invoked to get the response back to the client.
The Unary Server Interceptor looks fairly similar with the client, it is wraped within a function to evaluate the logging options if any. 

{{< code language="go" isCollapsed="false" >}}
func UnaryServerInterceptor(log logr.Logger, opts ...Option) grpc.UnaryServerInterceptor {
	o := evaluateServerOpt(opts)
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		request := req.(*api.ClientRequestType)
		log.Info("have received a request for " + request.GetVendor() + " as vendor ")
		startTime := time.Now()
		newCtx := newLoggerForCall(ctx, info.FullMethod, startTime)

		resp, err := handler(newCtx, req)

		if !o.shouldLog(info.FullMethod, err) {
			return resp, err
		}
		code := o.codeFunc(err)
		level := o.levelFunc(code)
		durField, durVal := o.durationFunc(time.Since(startTime))
		fields := Extract(newCtx)
		fields[durField] = durVal
		fields["grpc.code"] = code.String()

		levelLogf(log, level, "finished streaming call with code "+code.String(), fields, err)

		return resp, err
	}
}
{{< /code >}}

The main purpose of the **newLoggerForCall** function is to **Extract** the *[tag](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/3c51f7f332123e8be5a157c0802a228ac85bf9db/tags)* values within a map, modify it by adding the extra values and then recreate the context using the ToContext function. The new context will be used on calling the handler. In the post-processing phase, after the handler was called, the context is extracted again,the error is evaluated and the duration field is added to **fields**. All of these will be logged on the server side. Check out the **Run the Service** part to see the results.
> If you noticed [grpc_ctxtags](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/3c51f7f332123e8be5a157c0802a228ac85bf9db/tags) package was used for value extraction out of context. It adds a Tag object to the context that can be used by other middleware to add context about a request. Tags describe information about the request, and can be set and used by other middleware, or handlers. Tags are used for logging and tracing of requests. Tags are populated both upwards, *and* downwards in the interceptor-handler stack.

### Install the unary interceptor to Client and Server

#### Client

On the Client side a number of changes has been done in order to install the new created Interceptors. We instantiate the logger and select a log option, in this case DurationToTimeMillisField converts the duration to milliseconds and uses the key grpc.time_ms.

{{< code language="go" isCollapsed="false" >}}
logger := klogr.New()

opts := []grpcklog.Option{
		grpcklog.WithDurationField(grpcklog.DurationToTimeMillisField),
}
{{< /code >}}

A client connection can be configured by supplying **DialOption** functional option values to the **grpc.Dial** function. The WithUnaryInterceptor function returns a DialOption that specifies the interceptor for unary RPCs.

{{< code language="go" isCollapsed="false" >}}
func WithUnaryInterceptor(f UnaryClientInterceptor) DialOption
{{< /code >}}

We create the DialOptions and provide as argument to grpc.Dial function:

{{< code language="go" isCollapsed="false" >}}
dialOpts := []grpc.DialOption{
		grpc.WithUnaryInterceptor(grpcklog.UnaryClientInterceptor(logger, opts...)),
		grpc.WithTransportCredentials(creds),
	}

conn, err := grpc.DialContext(ctx, net.JoinHostPort(*addr, *port), dialOpts...)
{{< /code >}}

Last thing we comment out the log statement within the called method and leave the Interceptor to handle the logging.

#### Server

Similarly, on the server side, we instatiate the logging and select the log options, in this case we setup it with a custom duration to log field function.

{{< code language="go" isCollapsed="false" >}}
logger := klogr.New()

optsLog := []grpcklog.Option{
		grpcklog.WithDurationField(func(duration time.Duration) (key string, value interface{}) {
			return "grpc.time_ns", duration.Nanoseconds()
		}),
}
{{< /code >}}

The server connection is configured by supplying **ServerOption** functional option values to the **grpc.NewServer** function. Like for the client, this function takes an interceptor and return a ServerOption.

{{< code language="go" isCollapsed="false" >}}
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption
{{< /code >}}

However, only one unary interceptor can be installed. In our case, we use two interceptors. Therefore, we need a function that creates a single interceptor out of a chain of many interceptors. The **grpc_middleware** package provides convenient chaining methods. WithUnaryServerChain is a grpc.Server config option that accepts multiple unary interceptors and creates a single interceptor, this is its signature.

{{< code language="go" isCollapsed="false" >}}
func WithUnaryServerChain(interceptors ...grpc.UnaryServerInterceptor) grpc.ServerOption
{{< /code >}}

As Tags are used for logging requests we have to pass the unary and server-streaming methods with the WithFieldExtractor option

{{< code language="go" isCollapsed="false" >}}
grpc.NewServer(
	grpc_middleware.WithUnaryServerChain(
		grpc_ctxtags.UnaryServerInterceptor(grpc_ctxtags.WithFieldExtractor(grpc_ctxtags.CodeGenRequestFieldExtractor)),
		grpcklog.UnaryServerInterceptor(logger, optsLog...),
	),
	grpc.Creds(creds),
)
{{< /code >}}

Let's assume that we have identified the caller using authethication or other methods and we want to have these information within logs, then we can create a function which add the fields within the context.

{{< code language="go" isCollapsed="false" >}}
func addCustomerToctx(ctx context.Context) {
	clientID := uuid.Must(uuid.NewRandom()).String()
	grpcklog.AddFields(ctx, map[string]interface{}{"Name": "Customer-0367" + clientID[:4]})
}
{{< /code >}}

### Run the Service, with Unary Interceptors

Notice the structural information provided by the logs.

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes google
I0121 09:21:43.468720   24825 client_interceptor.go:18]  "msg"="requesting all product types from vendor: google"  
I0121 09:21:43.481720   24825 client_interceptor.go:65]  "msg"="Info - The call finished with code OK"  "details"={"SystemField":"grpc client","grpc.method":"GetVendorProdTypes","grpc.service":"api.ProdService","grpc.time_ms":12.948}
google cloud products type are:  compute storage

$ go run server.go 
2020/01/21 09:21:33 Serving gRPC on https://localhost:8080
I0121 09:21:43.481232   24702 server_interceptor.go:19]  "msg"="have received a request for google as vendor "  
I0121 09:21:43.481329   24702 server_interceptor.go:103]  "msg"="Info - finished streaming call with code OK"  "details"={"Name":"Customer-03677bff","SystemField":"grpc server","grpc.code":"OK","grpc.method":"GetVendorProdTypes","grpc.request.deadline":"2020-01-21T09:21:47+02:00","grpc.service":"api.ProdService","grpc.time_ns":15878,"peer.address":"127.0.0.1:52284"}
{{< /code >}}

## Stream Interceptor

Stream interceptor intercepts stream RPC calls. An implementation of a stream interceptor usually include pre-processing, and stream operation interception. Pre-processing is similar to unary interceptor. However, rather than doing the RPC method invocation and post-processing afterwards, stream interceptor intercepts the users' operation on the stream. 

### Stream Client Interceptor

StreamClientInterceptor intercepts the creation of ClientStream and has this signature:

{{< code language="go" isCollapsed="false" >}}
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
{{< /code >}}

Invoker from Unary Interceptor has been replaced with **streamer**, which is called by the interceptor to create a stream. **desc** is a streaming RPC service’s method specification.
We don't have direct access to request information in order to log the request. Therefore we define a new struct wrappedStream, which is embedded with a ClientStream. We implement the SendMsg method on wrappedStream to intercepts the operation on the embedded ClientStream.

{{< code language="go" isCollapsed="false" >}}
type wrappedStream struct {
	grpc.ClientStream
	logr.Logger
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	request := m.(*api.ClientRequestProds)
	w.Info("requesting all " + request.GetProductType() + "products from " + request.GetVendor())
	return w.ClientStream.SendMsg(m)
}

func newWrappedStream(s grpc.ClientStream, log logr.Logger) grpc.ClientStream {
	return &wrappedStream{
		s,
		log,
	}
}
{{< /code >}}

and the stream Client Interceptor.

{{< code language="go" isCollapsed="false" >}}
func StreamClientInterceptor(log logr.Logger, opts ...Option) grpc.StreamClientInterceptor {
	o := evaluateClientOpt(opts)
	return func(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
		fields := newClientLoggerFields(ctx, method)
		startTime := time.Now()
		clientStream, err := streamer(ctx, desc, cc, method, opts...)
		logFinalClientLine(o, log, fields, startTime, err, "finished client streaming call")
		return newWrappedStream(clientStream, log), err
	}
}
{{< /code >}}

The rest of the logic is similar to Unary Client Interceptor. Search above for the functionality explanation of the two functions newClientLoggerFields and logFinalClientLine.

### Stream Server Interceptor

Stream Server Interceptor has the signature:

{{< code language="go" isCollapsed="false" >}}
type StreamServerInterceptor func(srv interface{}, stream ServerStream, info *StreamServerInfo, handler StreamHandler) error
{{< /code >}}

Where **stream** defines the the server-side behavior of a streaming RPC and **info** provides various information about the streaming RPC on server side. **handler** is called by gRPC server to complete the execution of a streaming RPC.

{{< code language="go" isCollapsed="false" >}}
func StreamServerInterceptor(log logr.Logger, opts ...Option) grpc.StreamServerInterceptor {
	o := evaluateServerOpt(opts)
	return func(srv interface{}, stream grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
		startTime := time.Now()
		newCtx := newLoggerForCall(stream.Context(), info.FullMethod, startTime)
		wrapped := grpc_middleware.WrapServerStream(stream)
		wrapped.WrappedContext = newCtx

		err := handler(srv, wrapped)

		if !o.shouldLog(info.FullMethod, err) {
			return err
		}
		code := o.codeFunc(err)
		level := o.levelFunc(code)
		durField, durVal := o.durationFunc(time.Since(startTime))
		fields := Extract(newCtx)
		fields[durField] = durVal
		fields["grpc.code"] = code.String()

		levelLogf(log, level, "finished streaming call with code "+code.String(), fields, err)

		return err
	}
}
{{< /code >}}

It's not a surprise that this functions was implemented somehow similar to Unary Server Interceptor. There is a difference in the implementation. The context is not provided by the stream interceptor function, therefore a thin wrapper around grpc.ServerStream was created that allows modifying context, with the **grpc_middleware.WrapServerStream(stream)** function.

### Install the stream interceptor to Client and Server

#### Client

As the basics have been put in place by installing the UnaryClientInterceptor, this time we simply add the stream interceptor **StreamClientInterceptor** to the DialOption list.

{{< code language="go" isCollapsed="false" >}}
dialOpts := []grpc.DialOption{
		grpc.WithUnaryInterceptor(grpcklog.UnaryClientInterceptor(logger, opts...)),
		grpc.WithStreamInterceptor(grpcklog.StreamClientInterceptor(logger, opts...)),
		grpc.WithTransportCredentials(creds),
}
{{< /code >}}

WithStreamInterceptor returns a DialOption that specifies the interceptor for streaming RPCs.

{{< code language="go" isCollapsed="false" >}}
func WithStreamInterceptor(f StreamClientInterceptor) DialOption
{{< /code >}}

#### Server

We also install the **StreamServerInterceptor** on the server side.

{{< code language="go" isCollapsed="false" >}}
grpc_middleware.WithStreamServerChain(
			grpc_ctxtags.StreamServerInterceptor(grpc_ctxtags.WithFieldExtractor(grpc_ctxtags.CodeGenRequestFieldExtractor)),
			grpcklog.StreamServerInterceptor(logger, optsLog...),
),
{{< /code >}}

We could have used **StreamInterceptor**, which returns a ServerOption that sets the StreamServerInterceptor for the server.

{{< code language="go" isCollapsed="false" >}}
func StreamInterceptor(i StreamServerInterceptor) ServerOption
{{< /code >}}

But once again we use tags to pass the logging fields, therefore grpc_ctxtags.StreamServerInterceptor has to be installed as well. Because StreamInterceptor accepts only one function, we get use of **grpc_middleware.WithStreamServerChain**, which is a grpc.Server config option that accepts multiple stream interceptors.

{{< code language="go" isCollapsed="false" >}}
func WithStreamServerChain(interceptors ...grpc.StreamServerInterceptor) grpc.ServerOption
{{< /code >}}

### Run the Service, with Stream Interceptors

Notice the structural information provided by the logs.

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprods aws storage
I0131 09:22:46.383451    3909 client_interceptor.go:48]  "msg"="requesting all storage products from aws"
I0131 09:22:46.383423    3909 client_interceptor.go:81]  "msg"="Info - The call finished with code OK"  "details"={"SystemField":"grpc client","grpc.method":"GetVendorProds","grpc.service":"api.ProdService","grpc.time_ms":11.496}

Title: Amazon Aurora, Url: https://aws.amazon.com/rds/aurora/,  ShortUrl: https://made-up-url.com/0a24d7
Title: Amazon RDS, Url: https://aws.amazon.com/rds/,  ShortUrl: https://made-up-url.com/6b328b
Title: Amazon Redshift, Url: https://aws.amazon.com/redshift/,  ShortUrl: https://made-up-url.com/17857b
Title: Amazon DynamoDB, Url: https://aws.amazon.com/dynamodb/,  ShortUrl: https://made-up-url.com/3c3018
Title: Amazon ElastiCache for Memcached, Url: https://aws.amazon.com/elasticache/memcached/,  ShortUrl: https://made-up-url.com/c3d77f
Title: Amazon ElastiCache for Redis, Url: https://aws.amazon.com/elasticache/redis/,  ShortUrl: https://made-up-url.com/d3aaa9
Title: Amazon Neptune, Url: https://aws.amazon.com/neptune/,  ShortUrl: https://made-up-url.com/0794eb


$ go run server.go 
2020/01/21 09:22:39 Serving gRPC on https://localhost:8080
2020/01/21 09:22:46 have received a request for -> storage <- product type from -> aws <- vendor
I0121 09:22:46.019002   24950 server_interceptor.go:103]  "msg"="Info - finished streaming call with code OK"  "details"={"Name":"Customer-0367aeb1","SystemField":"grpc server","grpc.code":"OK","grpc.method":"GetVendorProds","grpc.request.deadline":"2020-01-21T09:22:50+02:00","grpc.service":"api.ProdService","grpc.time_ns":1640561,"peer.address":"127.0.0.1:52290"}

$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> storage <- product type from -> aws <- vendor
INFO:root:a number of 7 products were sent to client
{{< /code >}}

## Multiple Middlewares -- Opentelemetry

The goal of this section is to demonstrate that you can install multiple interceptors, that are taking care of a specific part of the service. You could have logging, monitoring, tracing, authentication and others working together. All of these will be processed from left to right, as they are installed, using the convenient chaining methods provided by grpc_middleware.
This time I'm not going to create my own middleware, but I'm going to use the one provided by [**opentelemetry-go**](https://github.com/open-telemetry/opentelemetry-go/tree/master/example/grpc). 

If you wonder what is Opentelemetry, check out [the presentation site](https://opentelemetry.io/). Based on their description, OpenTelemetry provides a single set of APIs, libraries, agents, and collector services to capture distributed traces and metrics from your application. You can analyze them using Prometheus, Jaeger, and other observability tools. It is a [CNCF Sandbox member](https://www.cncf.io/sandbox-projects/), formed through a merger of the [OpenTracing](https://opentracing.io/) and [OpenCensus](https://opencensus.io/) projects. The goal of OpenTelemetry is to provide a general-purpose API, SDK, and related tools required for the instrumentation of cloud-native software, frameworks, and libraries.

It's a cool tool and I highly recommend. Only the Unary interceptors are available at this point in time, if time permits maybe I'll try to see how can be implemented for streaming as well. I'll cover only a subset of information, therefore checkout the full implementation on [Github](https://github.com/danrusei/grpc_framework/tree/v0.5), and select the tag v0.5.

On the client side as well on the server side we first instantiate the telemetry.

{{< code language="go" isCollapsed="false" >}}
grpcopentelemetry.Init()
{{< /code >}}

It is a function that register a global trace provider. It needs an instance of trace provide **tp** which is created by the sdktrace.NewProvider function. The function takes a number of options as arguments. WithConfig sets the configuration to provider, WithSyncer appends the exporter to the existing list of Syncers.

{{< code language="go" isCollapsed="false" >}}
func Init() {
	exporter, err := stdout.NewExporter(stdout.Options{PrettyPrint: true})
	if err != nil {
		log.Fatal(err)
	}
	tp, err := sdktrace.NewProvider(
		sdktrace.WithConfig(sdktrace.Config{DefaultSampler: sdktrace.AlwaysSample()}),
		sdktrace.WithSyncer(exporter),
	)
	if err != nil {
		log.Fatal(err)
	}
	global.SetTraceProvider(tp)
}
{{< /code >}}

We know that grpc.WithUnaryInterceptor accepts only one interceptor and return a DialOption. To solve this we wrap both Unary client interceptors with grpc_middleware.ChainUnaryClient. Its job is to creates a single interceptor out of a chain of many interceptors.

{{< code language="go" isCollapsed="false" >}}
grpc.WithUnaryInterceptor(
	grpc_middleware.ChainUnaryClient(
		grpcklog.UnaryClientInterceptor(logger, opts...),
		grpcopentelemetry.UnaryClientInterceptor,
	),
),
{{< /code >}}

We add the opentelemetry components and call metadata.NewOutgoingContext(ctx, md), that creates a new context with outgoing metadata attached. 

{{< code language="go" isCollapsed="false" >}}
md := metadata.Pairs(
	"timestamp", time.Now().Format(time.StampNano),
	"client-id", "web-api-client-us-east-1",
	"user-id", "some-test-user-id",
)
ctx = metadata.NewOutgoingContext(ctx, md)
{{< /code >}}

On the server side, it's already configured with Interceptor server chain, so it's a matter of only adding the opentelemetry middleware.

{{< code language="go" isCollapsed="false" >}}
grpc_middleware.WithUnaryServerChain(
 	grpc_ctxtags.UnaryServerInterceptor(grpc_ctxtags.WithFieldExtractor(grpc_ctxtags.CodeGenRequestFieldExtractor)),
 	grpcklog.UnaryServerInterceptor(logger, optsLog...),
	grpcopentelemetry.UnaryServerInterceptor,
),
{{< /code >}}

I'm not going into details of how these Interceptors works, if you are interested, [check out the code](https://github.com/danrusei/grpc_framework/blob/v0.5/middleware/grpcopentelemetry/unary_interceptors.go). The main ideea is that the UnaryServerInterceptor intercepts and extracts incoming trace data and the UnaryClientInterceptor intercepts and injects outgoing trace.

### Run the Service, with Opentelemetry Interceptors

Notice both klog and opentracing working together as grpc middlewares.

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes oracle
I0127 11:36:11.034955   18076 client_interceptor.go:18]  "msg"="requesting all product types from vendor: oracle"  
{
	"SpanContext": {
		"TraceID": "503ae9aae593985ae5a96823ceff7503",
		"SpanID": "515e5ceec9c60fd5",
		"TraceFlags": 1
	},
	"ParentSpanID": "0000000000000000",
	"SpanKind": 1,
	"Name": "grpc_tracer/Cloud-Products-types",
	"StartTime": "2020-01-27T11:36:11.034975934+02:00",
	"EndTime": "2020-01-27T11:36:11.047985393+02:00",
	"Attributes": null,
	"MessageEvents": null,
	"Links": null,
	"Status": 0,
	"HasRemoteParent": false,
	"DroppedAttributeCount": 0,
	"DroppedMessageEventCount": 0,
	"DroppedLinkCount": 0,
	"ChildSpanCount": 0
}
I0127 11:36:11.048271   18076 client_interceptor.go:65]  "msg"="Info - The call finished with code OK"  "details"={"SystemField":"grpc client","grpc.method":"GetVendorProdTypes","grpc.service":"api.ProdService","grpc.time_ms":13.261}

oracle cloud products type are:  compute storage

$ go run server.go 
2020/01/27 11:32:55 Serving gRPC on https://localhost:8080
I0127 11:36:11.047142   17698 server_interceptor.go:19]  "msg"="have received a request for oracle as vendor "  
{
	"SpanContext": {
		"TraceID": "503ae9aae593985ae5a96823ceff7503",
		"SpanID": "4780a3c993489892",
		"TraceFlags": 1
	},
	"ParentSpanID": "515e5ceec9c60fd5",
	"SpanKind": 2,
	"Name": "grpc_tracer/Cloud-Products-types",
	"StartTime": "2020-01-27T11:36:11.047211115+02:00",
	"EndTime": "2020-01-27T11:36:11.04722305+02:00",
	"Attributes": [
		{
			"Key": "grpc.server",
			"Value": {
				"Type": "STRING",
				"Value": "api-server"
			}
		}
	],
	"MessageEvents": null,
	"Links": null,
	"Status": 0,
	"HasRemoteParent": true,
	"DroppedAttributeCount": 0,
	"DroppedMessageEventCount": 0,
	"DroppedLinkCount": 0,
	"ChildSpanCount": 0
}
I0127 11:36:11.047472   17698 server_interceptor.go:103]  "msg"="Info - finished streaming call with code OK"  "details"={"Name":"Customer-03678433","SystemField":"grpc server","grpc.code":"OK","grpc.method":"GetVendorProdTypes","grpc.request.deadline":"2020-01-27T11:36:15+02:00","grpc.service":"api.ProdService","grpc.time_ns":279834,"peer.address":"127.0.0.1:53234"}
{{< /code >}}

## REST over gRPC with grpc-gateway 

On these two articles I tried to highlight the benefits of using gRPC and why it drives high adoption lately. But the clients may not be prepared to migrate to a grpc client solution or they prefer REST services. With **grpc-gateway**, you can generate a reverse proxy that would translate REST into gRPC call via marshaling of the JSON request body into respective Go structures followed by the RPC endpoint call.

{{< image src="/img/2020/grpc-gateway.png" style="border-radius: 8px;" >}}

In addition to the packages that have been already installed, we need:

{{< code language="go" isCollapsed="false" >}}
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
{{< /code >}}

To build the gateway we add metadata to the ProdService proto to indicate that the GetVendorProdTypes & GetVendorProds RPC maps to a RESTful GET method with all RPC parameters mapped to a JSON body. Add a google.api.http annotation to your .proto file as well.

{{< code language="go" isCollapsed="false" >}}
import "google/api/annotations.proto";

service ProdService {
    rpc GetVendorProdTypes(ClientRequestType) returns (ClientResponseType) {
        option (google.api.http) = {
            get: "/api/prodtypes"
        };
    };
    rpc GetVendorProds(ClientRequestProds) returns (stream ClientResponseProds) {
        option (google.api.http) = {
            get: "/api/prods"
        };
    };
}
{{< /code >}}

I'm updating the Makefile and include --grpc-gateway_out argument to generate a reverse-proxy using protoc-gen-grpc-gateway.

{{< code language="bash" isCollapsed="false" >}}
generate:
	/home/rdan/protobuf/bin/protoc -I proto -I ${GOPATH}/pkg/mod/github.com/grpc-ecosystem/grpc-gateway@v1.12.2/third_party/googleapis --go_out=plugins=grpc:proto/ proto/api.proto
	/home/rdan/protobuf/bin/protoc -I proto -I ${GOPATH}/pkg/mod/github.com/grpc-ecosystem/grpc-gateway@v1.12.2/third_party/googleapis --grpc-gateway_out=logtostderr=true:proto/ proto/api.proto
{{< /code >}}

Running these commands it generates the api.pb.gw.go, besides the api.pb.go. Within api.pb.gw.go we get the function to register HTTP handler to call underlying gRPC service endpoints. The grpc.Serve() function as well as http.ListenAndServ are blocking functions, that returns only on error. Therefore we lunch two goroutines, one for the grpc server and the second for the reverse proxy.

{{< code language="go" isCollapsed="false" >}}
grpcAddr := fmt.Sprintf("localhost:%d", *portGRPC)
go func() {
	if err := runGRPCServer(logger, grpcAddr); err != nil {
		log.Fatalf("could not start the server: %s", err)
	}
}()

restAddr := fmt.Sprintf("localhost:%d", *portREST)
go func() {
	if err := runRESTServer(restAddr, grpcAddr); err != nil {
		log.Fatalf("could not start the server: %s", err)
	}
}()
{{< /code >}}

So I moved entire grpc configuration from run() to runGRPCServer(). The requests comming to the HTTP endpoint are proxied to the running gRPC service. **RegisterProdServiceHandlerFromEndpoint** registers the http handlers for service ProdService to "mux". The handlers forward requests to the grpc endpoint over "conn".

{{< code language="go" isCollapsed="false" >}}
func runRESTServer(restAddr, grpcAdd string) error {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux()
	creds, err := credentials.NewClientTLSFromFile("../cert/service.pem", "")
	if err != nil {
		return fmt.Errorf("could not load TLS certificate: %s", err)
	}
	// Setup the client gRPC options
	opts := []grpc.DialOption{grpc.WithTransportCredentials(creds)}
	// Register 
	err = api.RegisterProdServiceHandlerFromEndpoint(ctx, mux, grpcAdd, opts)
	if err != nil {
		return fmt.Errorf("could not register service ProdService: %s", err)
	}
	log.Printf("starting HTTP/1.1 REST server on %s", restAddr)
	http.ListenAndServe(restAddr, mux)
	return nil
}
{{< /code >}}

The working script that includes grpc-gateway configuration can be downloaded [from here](https://github.com/danrusei/grpc_framework).

### Run the REST Service

#### Client -- > Server

{{< code language="bash" isCollapsed="false" >}}
$ curl -X GET 'http://localhost:8081/api/prodtypes?vendor=oracle'
{"productType":" compute storage"}

$ go run server.go 
2020/01/27 16:41:40 Entering infinite loop
2020/01/27 16:41:40 starting HTTP/1.1 REST server on localhost:8081
2020/01/27 16:41:40 Serving gRPC on https://localhost:8080
I0127 16:42:27.447081   31394 server_interceptor.go:19]  "msg"="have received a request for oracle as vendor "  

...

I0127 16:42:27.447348   31394 server_interceptor.go:103]  "msg"="Info - finished streaming call with code OK"  "details"={"Name":"Customer-03674bcf","SystemField":"grpc server","grpc.code":"OK","grpc.method":"GetVendorProdTypes","grpc.service":"api.ProdService","grpc.start_time":"2020-01-27T16:42:27+02:00","grpc.time_ns":228493,"peer.address":"127.0.0.1:55346"}
{{< /code >}}

#### Client -- > Server -- > Storage

{{< code language="bash" isCollapsed="false" >}}
$ curl -X GET 'http://localhost:8081/api/prods?vendor=google&productType=compute'
{"result":{"product":{"title":"Compute Engine","url":"https://cloud.google.com/compute/","shortUrl":"https://made-up-url.com/7d62d1"}}}
{"result":{"product":{"title":"App Engine","url":"https://cloud.google.com/appengine/","shortUrl":"https://made-up-url.com/ccf764"}}}
{"result":{"product":{"title":"Cloud Functions","url":"https://cloud.google.com/functions/","shortUrl":"https://made-up-url.com/302ac9"}}}
{"result":{"product":{"title":"Cloud Run","url":"https://cloud.google.com/run/","shortUrl":"https://made-up-url.com/b28804"}}}
{"result":{"product":{"title":"GKE","url":"https://cloud.google.com/kubernetes-engine/","shortUrl":"https://made-up-url.com/5ae835"}}}

$ go run server.go 
2020/01/27 16:43:57 Entering infinite loop
2020/01/27 16:43:57 starting HTTP/1.1 REST server on localhost:8081
2020/01/27 16:43:57 Serving gRPC on https://localhost:8080
2020/01/27 16:44:15 have received a request for -> compute <- product type from -> google <- vendor
I0127 16:44:15.299347   31615 server_interceptor.go:103]  "msg"="Info - finished streaming call with code OK"  "details"={"Name":"Customer-03679b88","SystemField":"grpc server","grpc.code":"OK","grpc.method":"GetVendorProds","grpc.service":"api.ProdService","grpc.start_time":"2020-01-27T16:44:15+02:00","grpc.time_ns":1476858,"peer.address":"127.0.0.1:55376"}

$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> compute <- product type from -> google <- vendor
INFO:root:a number of 5 products were sent to client
{{< /code >}}

## Conclusion

Although REST is still the standard in the industry, gRPC is gaining a lot of ground due to high adoption of microservices.
gRPC is a modern, high performant RPC framework that integrates alot of functionality required for modern workloads. It is more efficient and faster comparing to the traditional REST service as it leverage http/2 for transport and protocol buffers for binary serialization.
