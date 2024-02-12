---
title: "Part 1: Demystifying gRPC"
date: 2020-01-06
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

As we are moving more towards distributed systems and micro-services, the way the services are communicating between each other becomes increasingly important. As you can imagine, the request-response delay is multiplied by the number micro-services running in the backend. Until recently, the industry standard was JSON over REST, which is a software architectural style that defines a set of recommendations for designing loosely coupled applications that use the HTTP protocol for data transmission. REST architecture allows API providers to deliver data in multiple formats, JSON is currently the preferred one.

Google introduced a new standard for micro-services communications, named gRPC. If you are searching on the web for grpc vs rest, you'll find plenty of articles discussing the benefits of each other with increased attention to performance. Nowadays, multiple companies have migrated from Rest to gRPC which indicates that the value added by this protocol for communication worthwhile. It is beyond the scope of this article to compare the two protocols, if you are interested in this, there are multiple resources available. In this article I would like to give you a taste of what is gRPC, how it can be used, and showcase some beyond the basic capabilities.

This is Part 1 of a series of three articles describing gRPC functionality. If you want to go beyond the basics, check out [Part 2](https://dev-state.com/posts/grpc_framework_2/) & [Part 3](https://dev-state.com/posts/grpc_framework_3/)


## What is gRPC

gRPC (gRPC Remote Procedure Calls) is an open source remote procedure call (RPC) system. In gRPC a client application can directly call methods on a server application on a different machine as if it were a local object, making it easier to create distributed applications and services. As in many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, the client has a stub (referred to as just a client in some languages) that provides the same methods as the server. Worth to mention that it uses HTTPv2 based transport and support bi-directional streaming.

The client or server do not need to be written in the same language, but they are created using a common definition file which define the methods that can be called remotely with their parameters and return types. It uses [protocol buffers](https://developers.google.com/protocol-buffers/) as the Interface Definition Language. Protocol buffers is language-neutral, you define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. JSON is not very efficient to parse for a computer even though it’s more human-readable, computers parse binary quicker. Protocol buffers are encoding the messages in a binary wire format. 

In this article I'll showcase how to generate the gRPC client and server interfaces from the .proto service definition in GO and Python.

## Simple gRPC service

I'll walk you through a simple service example, covering the following aspects:

* Define a service in a .proto file.
* Generate server and client code using the protocol buffer compiler.
* Use the Go gRPC API to write a simple client and server for your service.

The complete code for this part can be found [HERE](https://github.com/danrusei/grpc_framework/tree/v0.1).

The application I'm creating here it is named "Cloud Products" and it's aiming to provide to the client the details about the cloud portfolio products of some major cloud providers. It is not a full featured App, the idea is just to showcase the gRPC capabilities. I'll start with a simple gRPC client - server and introduce new concepts moving on. 

First step is to define the gRPC service, specifying the methods and the request / response types. This is a simple RPC where the client sends a request to the server using the stub and waits for a response to come back, just like a normal function call. In our example the method is **GetVendorProdTypes** and the **ClientRequestType** and **ClientResponseType** are protocol buffer message type definitions. 
As you can see, each field in the message definition has a unique number. These numbers are used to identify the fields in the message binary format, and should not be changed once your message type is in use. This explains why protocol buffers are extensible mechanism for serializing structured data. Let's assume that you need to add another field to your data after the schema is already in use. Because you explicitly give it a number, your deserializer is still able to load data serialized with the old numbering scheme, ignoring deserialization of non-existent data.

{{< code language="proto" isCollapsed="false" >}}
syntax = "proto3";

package api;

service ProdService {
    rpc GetVendorProdTypes(ClientRequestType) returns (ClientResponseType);
}

message ClientRequestType {
    string vendor = 1;
}

message ClientResponseType {
    string productType = 1;
}
{{< /code >}}

Next we need to generate the gRPC client and server interfaces from our .proto service definition. We do this by using the protocol buffer compiler protoc with the special gRPC Go plugin.
Protoc compiler can be downloaded and installed from here https://github.com/protocolbuffers/protobuf/releases. You need also to install the grpc package and protoc plugin for go.

{{< code language="bash" isCollapsed="false" >}}
 go get -u google.golang.org/grpc
 go get -u github.com/golang/protobuf/protoc-gen-go
{{< /code >}}

Then run the protoc executable, as below. For easy usage I have created a **Makefile** in the root of the directory, just modify the path to the protoc binary file.

{{< code language="bash" isCollapsed="false" >}}
protoc -I proto --go_out=plugins=grpc:proto/ proto/api.proto
{{< /code >}}

The **api.pb.go** generated file contain:

* All the protocol buffer code to populate, serialize, and retrieve our request and response message types
* An interface type (or stub) for clients to call with the methods defined in the **ProdService** service.
* An interface type for servers to implement, also with the methods defined in the **ProdService** service.

### Create the Server

Two things have to be done to make the server respond to RPC calls:

* Implement the service interface generated from our service definition.
* Run a gRPC server and listen for requests from clients 

Looking to **api.pb.go** generated file the **GetVendorProdTypes** has to be implemented on the server side.

{{< code language="go" isCollapsed="false" >}}
// ProdServiceServer is the server API for ProdService service.
type ProdServiceServer interface {
	GetVendorProdTypes(context.Context, *ClientRequestType) (*ClientResponseType, error)
}
{{< /code >}}

The **server** struct implements the method, therefore it satify the **ProdServiceServer** interface. The method is passed a context object for the RPC and the client’s **ClientRequestType** protocol buffer request. It returns a **ClientResponseType** protocol buffer object and an error. 
The server struct has a field **prodTypes** which is a map containing the Cloud provider as key and the products as slices of strings. **req.GetVendor()** is a getter implemented to retrieve the **Vendor** value from the **ClientRequestType** struct.

{{< code language="go" isCollapsed="false" >}}
type server struct {
	prodTypes map[string][]string
}

//GetVendorProdTypes implement the GRPC server function
func (serv server) GetVendorProdTypes(ctx context.Context, req *api.ClientRequestType) (*api.ClientResponseType, error) {

	log.Printf("have received a request for -> %s <- as vendor", req.GetVendor())

	var prodTypes string

	if vendorProdTypes, found := serv.prodTypes[req.GetVendor()]; found {

		for _, prodType := range vendorProdTypes {
			prodTypes = prodTypes + " " + prodType
		}

	} else {
		return nil, fmt.Errorf("wrong vendor, select between google, aws, oracle")
	}

	clientResponse := api.ClientResponseType{
		ProductType: prodTypes,
	}

	return &clientResponse, nil
}
{{< /code >}}

To build and start the server , we need to:

* Listen to client requests using net.Listen()
* Create an instance of the gRPC server using grpc.NewServer().
* Register our service implementation with the gRPC server.
* Call Serve() on the server to do a blocking wait until the process is killed or stoped.

{{< code language="go" isCollapsed="false" >}}
lis, err := net.Listen("tcp", addr)
if err != nil {
	return fmt.Errorf("could not listen on the port %s: %s", addr, err)
}

srv := grpc.NewServer()
    
//Register our service implementation with the gRPC server, newServer is a constructor for server struct,
//which implements ProdServiceServer interface
api.RegisterProdServiceServer(srv, newServer(vendorServices))

log.Printf("Serving gRPC on https://%s", addr)

if err := (srv.Serve(lis)); err != nil {
	return fmt.Errorf("Unable to start GRPC server: %s", err)
}

return nil
{{< /code >}}

### Create the Client

In order to create the gRPC client, we need:

* to create a gRPC channel to communicate with the server, using grpc.Dial() with correct server address and port
* to create a client stub to perform RPCs, using NewProdServiceClient, which instantiate a ProdServiceClient, a client API for ProdService service

The **addr** and **port** are parsed from flags with the defaults localhost:8080 for our test instance.

{{< code language="go" isCollapsed="false" >}}
conn, err := grpc.Dial(net.JoinHostPort(*addr, *port), grpc.WithInsecure())
if err != nil {
	log.Fatalf("Failed to dial server:, %s", err)
}
defer conn.Close()

client := api.NewProdServiceClient(conn)
{{< /code >}}

To make the client extensible, in the main function, I'm using a switch statement to call different methods. Once the script is run the received arguments are checked.

{{< code language="go" isCollapsed="false" >}}
ctx := context.Background()

switch cmd := flag.Arg(0); cmd {
case "getprodtypes":
	err = getprodtypes(ctx, client, flag.Arg(1))
default:
	err = fmt.Errorf("unknown subcommand %s", cmd)
}
if err != nil {
	fmt.Fprintln(os.Stderr, err)
	os.Exit(1)
}
{{< /code >}}

We can call the service methods like calling a local method using **client.GetVendorProdTypes**. **response.GetProductType()** is a getter implemented to retrieve the **ProductType** value out of the **ClientResponseType** struct.

{{< code language="go" isCollapsed="false" >}}
func getprodtypes(ctx context.Context, client api.ProdServiceClient, vendor string) error {

	if vendor == "" {
		return fmt.Errorf("Vendor arg is missing, select between available cloud vendors: google, aws, oracle")
	}

	requestProd := api.ClientRequestType{
		Vendor: vendor,
	}

	response, err := client.GetVendorProdTypes(ctx, &requestProd)
	if err != nil {
		return fmt.Errorf("Could not get the products: %v", err)
	}

	fmt.Printf("%s cloud products type are: %s\n", vendor, response.GetProductType())

	return nil

}
{{< /code >}}

### Run the service

[The V0.1](https://github.com/danrusei/grpc_framework/tree/v0.1) tag has the above code.
This simple example it will be used as a base to the next steps. But first let's run it and see how it works.


Run the Server:
{{< code language="bash" isCollapsed="false" >}}
go run server.go 
2020/01/04 15:27:35 Serving gRPC on https://localhost:8080
2020/01/04 15:27:39 have received a request for -> sdss <- as vendor
2020/01/04 15:27:52 have received a request for -> google <- as vendor
{{< /code >}}

Run the Client Side (check out the errors if the command is incomplete or wrong):

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go 
missing command: getprodtypes
exit status 1

$ go run client.go getprodtypes
Vendor arg is missing, select between available cloud vendors: google, aws, oracle
exit status 1

$ go run client.go getprodtypes sdss
rpc error: code = InvalidArgument desc = error while calling client.GetVendorProdTypes() method: wrong vendor, select between google, aws, oracle
exit status 1

$ go run client.go getprodtypes google
google cloud products type are:  compute storage
{{< /code >}}

## gRPC Errors and Service Cancelation

The Go gRPC implementation has a [**status**](https://godoc.org/google.golang.org/grpc/status) package which exposes a simple interface for creating rich gRPC errors. These errors are serialized and transmitted on the wire between server and client. Package [**codes**](https://godoc.org/google.golang.org/grpc/codes) defines the canonical error codes used by gRPC. It is consistent across various languages.
To send an error, return status.Errorf with error message and code:

{{< code language="go" isCollapsed="false" >}}
status.Errorf(grpc error code, error message)
{{< /code >}}

On the client side you can check the errors with:

{{< code language="go" isCollapsed="false" >}}
st, ok := status.FromError(err)
if !ok {
    // Error was not a status error
}
{{< /code >}}

In our case, we check if the vendor information was sent, if not we wrap the error within a gRPC status error.

{{< code language="go" isCollapsed="false" >}}
if vendorProdTypes, found := serv.prodTypes[req.GetVendor()]; found {

 ...........do somtheing................

	}
} else {
	return nil, status.Errorf(codes.InvalidArgument, "wrong vendor, select between google, aws, oracle")
}
{{< /code >}}

and this how we check the errors on the client side:

{{< code language="go" isCollapsed="false" >}}
if err != nil {
		if errStatus, ok := status.FromError(err); ok {
			return status.Errorf(errStatus.Code(), "error while calling client.GetVendorProdTypes() method: %v ", errStatus.Message())
		}
		return fmt.Errorf("Could not get the products: %v", err)
	}
{{< /code >}}

#### Deadline exceeded

Let's assume that the server has to do some heavy processing to return the values and the client is waiting only for defined period of time. We could add context deadline on the client side and emulate heavy load with **sleep** command on the server side. For that, replace context.Background() with:

{{< code language="go" isCollapsed="false" >}}
ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()
{{< /code >}}

Assuming that it takes more than 4 seconds for the server to process the data and reply back, this is what happens:

Client Side:

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes oracle
Could not get the products: rpc error: code = DeadlineExceeded desc = context deadline exceeded
exit status 1
{{< /code >}}

Server Side:

{{< code language="bash" isCollapsed="false" >}}
$ go run server.go 
2020/01/04 16:12:32 Serving gRPC on https://localhost:8080
2020/01/04 16:12:37 have received a request for -> oracle <- as vendor
2020/01/04 16:12:43 the response is sent to client:  compute storage
{{< /code >}}

But there is a small problem, as you can see from the server logs, the server continue the work and it is attempting to send over the response over the wire.
Which is not the best case, as we continue to consume the server resources even if nobody needs the response anymore. In order to solve this, we can modify the server to cancel the work once deadline exceeded. The error is propagated by the context package.

{{< code language="go" isCollapsed="false" >}}
if ctx.Err() == context.DeadlineExceeded {
		log.Printf("dealine has exceeded, stoping server side operation")
		return nil, status.Error(codes.DeadlineExceeded, "dealine has exceeded, stoping server side operation")
	}
{{< /code >}}

Let's check this again, as you can see, this time the server logged "exceeded, stopping server side operation", which is exactly what is expected.

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes oracle
Could not get the products: rpc error: code = DeadlineExceeded desc = context deadline exceeded
exit status 1
{{< /code >}}

The result on server side:

{{< code language="bash" isCollapsed="false" >}}
$ go run server.go 
2020/01/04 16:24:46 Serving gRPC on https://localhost:8080
2020/01/04 16:24:52 have received a request for -> oracle <- as vendor
2020/01/04 16:24:58 dealine has exceeded, stoping server side operation
{{< /code >}}

#### Request is Canceled

However, there are cases when the user is not waiting for a response and the request is canceled. Unless we code this explicitly, the server don't know about this and will continue the work, which again it is not efficient. But, we can check again the context for specific canceled error and if it's the case we stop the process.

{{< code language="go" isCollapsed="false" >}}
if ctx.Err() == context.Canceled {
		log.Print("the user has canceled the request, stoping server side operation")
		return nil, status.Error(codes.Canceled, "the user has canceled the request, stoping server side operation")
	}
{{< /code >}}

As you can see from the code below, the server stops the work this time.

client side:

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes oracle
^Csignal: interrupt
{{< /code >}}

server side:

{{< code language="bash" isCollapsed="false" >}}
$ go run server.go 
2020/01/04 17:06:22 Serving gRPC on https://localhost:8080
2020/01/04 17:06:26 have received a request for -> oracle <- as vendor
2020/01/04 17:06:32 the user has canceled the request, stoping server side operation
{{< /code >}}

## Securing gRPC connections with SSL/TLS

The primary goal of the Transport Layer Security (TLS) protocol is to provide privacy and data integrity between two communicating applications. The TLS Handshake Protocol, allows the server and client to authenticate each other and to negotiate an encryption algorithm and cryptographic keys before the application protocol transmits or receives its first byte of data. gRPC is designed to work with a variety of authentication mechanisms, making it easy to safely use gRPC to talk to other systems. SSL/TLS is a supported mechanism, or you can plug in your own authentication system.

It is out of the scope of this tutorial to explain in details the authentication mechanism, below it is how we generate the keys and certificate using openssl commands. While an SSL Certificate is most reliable when issued by a trusted Certificate Authority (CA), we will be using self-signed certificates for the purpose of this post, meaning we sign them ourselves.

{{< code language="bash" isCollapsed="false" >}}
#Create Root signing Key
$ openssl genrsa -out ca.key 4096

#Generate self-signed Root certificate
$ openssl req -new -x509 -key ca.key -sha256 -subj "/C=RO/ST=IL/O=BUH, Inc." -days 365 -out ca.cert

#Create a Key certificate for the Server
$ openssl genrsa -out service.key 4096

#Create a signing CSR
$ openssl req -new -key service.key -out service.csr

#Generate a certificate for the Server
$ openssl x509 -req -sha256 -in service.csr -CA ca.cert -CAkey ca.key -CAcreateserial -out service.pem -days 365 -sha256
{{< /code >}}

To make the secure connection between server and client, the server needs to be initialized with a public/private key pair and the client needs to have the server’s public key. **NewServerTLSFromFile** constructs TLS credentials from the input certificate file and key file for server.  **grpc.NewServer** function definition is "func NewServer(opt ...ServerOption) Server". It means that it accepts a variable number of grpc.ServerOption values. **grpc.Creds** returns a ServerOption that sets credentials for server connections.

{{< code language="go" isCollapsed="false" >}}
creds, err := credentials.NewServerTLSFromFile("../cert/service.pem", "../cert/service.key")
if err != nil {
	return fmt.Errorf("could not process the credentials: %v", err)
}

// Create an array of gRPC options with the credentials
opts := []grpc.ServerOption{grpc.Creds(creds)}

srv := grpc.NewServer(opts...)
{{< /code >}}

On the client side we supply a **grpc.WithInsecure()** value to the **grpc.Dial** function. The grpc.WithInsecure() function returns a DialOption value which disables transport security for the client connection. 
But as we want to move to a secure connection, we use **NewClientTLSFromFile** to construct the TLS credentials from the input certificate file for client. **grpc.WithTransportCredentials** returns a DialOption that configures the connection level TLS/SSL security credentials.

{{< code language="go" isCollapsed="false" >}}
creds, err := credentials.NewClientTLSFromFile("../cert/service.pem", "")
if err != nil {
	log.Fatalf("could not process the credentials: %v", err)
}

conn, err := grpc.Dial(net.JoinHostPort(*addr, *port), grpc.WithTransportCredentials(creds))
if err != nil {
	log.Fatalf("Failed to dial server:, %s", err)
}
defer conn.Close()
{{< /code >}}

We just scratch the surface of the authethication mechanisms, there are plenty of examples on the web that goes in much detail on using CA certificates and Mutual TLS. Definetely you need a better mechanism to spread the keys across the vast amount of micro-services. A good starting resource to understand how to create secure conection in Go is Liz Rice's [talk on youtube](https://www.youtube.com/watch?v=kxKLYDLzuHA&list=WL&index=7&t=527s).

[The V0.2](https://github.com/danrusei/grpc_framework/tree/v0.2) tag extend previous code with gRPC error handlng and SSL/TLS connection.
