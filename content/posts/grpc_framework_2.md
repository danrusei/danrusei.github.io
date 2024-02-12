---
title: "Part 2: Demystifying gRPC"
date: 2020-01-10
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

Let's assume that the dev teams have been asked to extend the services and there are multiple teams working on it. Often, the teams prefer a specific programming language for a certain job. Luckily, gRPC is supported by a broad range of languages. We are going to extend the service to communicate to a storage service, but this time the service will be written in Python. This simple service can be written in whatever language of choice but in order to showcase the gRPC functionality in Python we could say that it is a project specification.

This is Part 2 of a series of three articles describing gRPC functionality. I would recommend to read [Part 1](https://dev-state.com/posts/grpc_framework_1/) as this is the continuation.

[The V0.3](https://github.com/danrusei/grpc_framework/tree/v0.3) tag contain the code for the extended service and the gRPC Service Storage in Python.

## Extend the service

We update first the **api.proto** file, to include the new rpc procedures. The old method **GetVendorProdTypes** remains the same, with the same message types. **GetVendorProds** rpc has been added, which allow the client to interrogate the server for detailed products for a selected **productType** and from a selected **vendor**. The communication between the **server** and **storage** is defined by another service named **StorageService**, that includes the rpc method **GetProdsDetail**.

{{< code language="proto" isCollapsed="false" >}}
syntax = "proto3";

package api;

service ProdService {
    rpc GetVendorProdTypes(ClientRequestType) returns (ClientResponseType);
    rpc GetVendorProds(ClientRequestProds) returns (stream ClientResponseProds);
}

message ClientRequestType {
    string vendor = 1;
}

message ClientResponseType {
    string productType = 1;
}

message ClientRequestProds {
    string vendor = 1;
    string productType = 2;
}

message ClientResponseProds {
    ProdsPrep product = 1;
}

message ProdsPrep {
    string title = 1;
    string url = 2;
    string shortUrl = 3;
}

service StorageService {
    rpc GetProdsDetail(StorageRequest) returns (StorageResponse);
}

message StorageRequest {
    string vendor = 1;
    string productType = 2;
}

message StorageResponse {
    repeated Product prodDetail = 1;
}

message Product {
    string title = 1;
    string url = 2;
}
{{< /code >}}

The idea is quite simple, the client has the option to make two type of requests. One to get the product types offered by the vendor, as an example AWS offer Compute and Storage type of products. Second is to retrieve the products details, belonging to the selected product type. 
In other words the client can request:

{{< code language="bash" isCollapsed="false" >}}
$ go run client.go getprodtypes aws
aws cloud products type are:  compute storage

and

$ go run client.go getprods aws storage
Title: Amazon Aurora, Url: https://aws.amazon.com/rds/aurora/,  ShortUrl: https://made-up-url.com/2a2075
Title: Amazon RDS, Url: https://aws.amazon.com/rds/,  ShortUrl: https://made-up-url.com/6402f0
Title: Amazon Redshift, Url: https://aws.amazon.com/redshift/,  ShortUrl: https://made-up-url.com/f109d2
Title: Amazon DynamoDB, Url: https://aws.amazon.com/dynamodb/,  ShortUrl: https://made-up-url.com/6bbdbc
Title: Amazon ElastiCache for Memcached, Url: https://aws.amazon.com/elasticache/memcached/,  ShortUrl: https://made-up-url.com/1c62ae
Title: Amazon ElastiCache for Redis, Url: https://aws.amazon.com/elasticache/redis/,  ShortUrl: https://made-up-url.com/820eee
Title: Amazon Neptune, Url: https://aws.amazon.com/neptune/,  ShortUrl: https://made-up-url.com/9b42ef
{{< /code >}}

As you can see the second response includes all the fields defined in **ProdsPrep** proto message. Once the client request the products, it receive a reply with: the product **title** , the **url** to the product page and for convenience a **shorturl**, which is a fake one. 

The flow:

**GO Cient** <-- (gRPC) --> **GO API Server** <-- (gRPC) --> **Backend Python Storage**

Behind the scene:

* the request is sent by the client, including the mandatory args: **vendor** name and the **product type**
* the request is received by the **api server**, which calls a RPC function of the **Storage** service
* the **Storage** service is a Python gRPC server. It responds back to the **api server** with a **list** of products, that has a **Title** and a **Url**
* the **Api Server** receive the response, and modify it by adding a **ShortUrl**.  The answer is streamed back to the GO client. 

There are two things you may observe in the above protobuf code. A **repeated** field that is translated in a GO slice and a Python list once the files are generated. And second is the **stream** word within this rpc signature "rpc GetVendorProds(ClientRequestProds) returns (stream ClientResponseProds);", of which I will talk more later.

## Create the Python GRPC server (Storage)

The main purpose of the Storage service is to serve the requesters with a list of products. As this is a toy project, the products are retrieved from memory, more specific they are populated within a list of dictionaries. Check out the [python script](https://github.com/danrusei/grpc_framework/blob/master/storage/storage.py) and find the data created at the top of the file. In a production environment there will be a real database.

### Generate client and server code

Next we generate the gRPC client and server interfaces from the .proto service definition, using following command:

{{< code language="bash" isCollapsed="false" >}}
python3 -m grpc_tools.protoc -I./proto --python_out=./proto --grpc_python_out=./proto ./proto/api.proto
{{< /code >}}

This has been added to [Makefile](https://github.com/danrusei/grpc_framework/blob/master/Makefile). The Python files generated are named **api_pb2.py** and **api_pb2_grpc.py** and contain:

* classes for the messages defined in api.proto
    * api_pb2.StorageRequest for the request format
    * api_pb2.StorageResponse for the response format
* classes for the service defined in api.proto:
    * StorageServiceStub, which can be used by clients to invoke StorageService RPCs
    * StorageServiceServicer, which defines the interface for implementations of the StorageService service
* the functions for the service defined in api.proto
    * add_StorageServiceServicer_to_server, which adds a StorageServiceServicer to a grpc.Server
    * GetProdsDetail method which we'll have to implement, this is where the business logic resides 

### Create the Python server

Creating and running the StorageService has following items:

* Implement the servicer interface generated from our service definition with functions that perform the actual “work” of the service. 
* Run a gRPC server to listen for requests from clients and transmit responses.

The **Storage** class implements the **StorageService** methods, it our case is only **GetProdsDetail**. It is passed **api__pb2.StorageRequest** request for the RPC, and a **grpc.ServicerContext** object that provides RPC-specific information such as timeout limits. It returns a **api_pb2.StorageResponse** response.

Notice the **try/except** code, that returns an error if the provided **vendor** and **product_type** aren't valid. If an error occurs, gRPC returns one of its error status codes instead, with an optional string error message that provides further details about what happened. Error information is available to gRPC clients in all supported languages.

The context.is_active() describes whether the RPC is active or has terminated. If it is active we create **products** that contain a list of **api_pb2.Product()** objects and send it over the line by returning **api_pb2.StorageResponse(prodDetail=products)**.

{{< code language="python" isCollapsed="false" >}}
class Storage(api_pb2_grpc.StorageServiceServicer):      

    def GetProdsDetail(self, request, context):

        # Retrieve vendor and prodType from client
        vendor = request.vendor.lower()
        product_type = request.productType.lower()

        logging.info("have received a request for -> {} <- product type from -> {} <- vendor".format(product_type, vendor))

        try:
            prod_type_list = get_prods(vendor, product_type)
        except AssertionError as error:
            print(error)
            context.set_details(error)
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            return api_pb2.StorageResponse()
        
       # time.sleep(5)
        products = []

        if context.is_active():   
            for prod in prod_type_list:
                product = api_pb2.Product()
                product.title = prod["title"]
                product.url = prod["url"]
                products.append(product)
        
        else:
        context.set_details(error)
            context.set_code(grpc.StatusCode.DEADLINE_EXCEEDED)
            logging.info("the connection to {} has dropped".format(context.peer()))
            return api_pb2.StorageResponse()
        
        logging.info("a number of {} products were sent to client".format(len(products)))

        return api_pb2.StorageResponse(prodDetail=products)
{{< /code >}}

Once we have implemented the method, next step is to start up a gRPC server so that clients can actually use the service. Because start() does not block you may need to sleep-loop if there is nothing else for your code to do while serving.

{{< code language="python" isCollapsed="false" >}}
def serve(port):
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    api_pb2_grpc.add_StorageServiceServicer_to_server(Storage(), server)
    server.add_insecure_port('[::]:' + str(port))
    server.start()
    print("Listening on port {}..".format(port))
    try:
        while True:
            time.sleep(10000)
    except KeyboardInterrupt:
        server.stop(0)

if __name__== "__main__":
    logging.basicConfig()
    serve(6000)
{{< /code >}}

Here is [the Link](https://github.com/danrusei/grpc_framework/blob/master/storage/storage.py) to the Storage (Python gRPC) final code.

## Extend the GO API server to support GetVendorProds RPC method

The function below is created on the GO Server API to satisfy the **ProdServiceServer** interface. It aims to get the request from the client and send over to the backend system to retrieve the required data. The data is augmented by adding the ShortUrl field and stream over to the client. Notice that we take the context from the stream.Context() and pass it over to the backend function RPC call.
Instead of getting a simple request and response objects in our method parameters, this time we get a request object (the api.ClientRequestProds) and a special **ProdService_GetVendorProdsServer** object to write our responses.
In the method, we populate as many api.ClientResponseProds objects as we need to return, writing them to the ProdService_GetVendorProdsServer using its Send() method. Finally, we return a nil error to tell gRPC that we’ve finished writing responses. Should any error happen in this call, we return a non-nil error, the gRPC layer will translate it into an appropriate RPC status to be sent on the wire.

{{< code language="go" isCollapsed="false" >}}
func (serv *server) GetVendorProds(req *api.ClientRequestProds, stream api.ProdService_GetVendorProdsServer) error {

	log.Printf("have received a request for -> %s <- product type from -> %s <- vendor", req.GetProductType(), req.GetVendor())

	conn, err := grpc.Dial(net.JoinHostPort(serv.storageAddr, serv.storagePort), grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Failed to dial server:, %s", err)

	}
	defer conn.Close()

	ctx := stream.Context()

	client := api.NewStorageServiceClient(conn)
	response, err := client.GetProdsDetail(ctx, &api.StorageRequest{
		Vendor:      req.GetVendor(),
		ProductType: req.GetProductType(),
	})
	if err != nil {
		if errStatus, ok := status.FromError(err); ok {
			log.Printf("error while calling client.GetProdsDetail() method: %v ", errStatus.Message())
			return status.Errorf(errStatus.Code(), "error while calling client.GetProdsDetail() method: %v ", errStatus.Message())
		}
		log.Printf("error while calling client.GetProdsDetail() method: %v ", err)
		return status.Errorf(codes.Internal, "error while calling client.GetProdsDetail() method: %v ", err)
	}

	for _, prod := range response.ProdDetail {

		id := uuid.Must(uuid.NewRandom()).String()

		if err := stream.Send(&api.ClientResponseProds{
			Product: &api.ProdsPrep{
				Title:    prod.GetTitle(),
				Url:      prod.GetUrl(),
				ShortUrl: "https://made-up-url.com/" + id[:6],
			},
		}); err != nil {
			return status.Error(codes.Internal, "not able to send the response")
		}

		// to simulate heavy processing **increase it ** -- to test out DeadlineExceeded
		//time.Sleep(1 * time.Second)

		if ctx.Err() == context.DeadlineExceeded {
			log.Printf("dealine has exceeded, stoping server side operation")
			return status.Error(codes.DeadlineExceeded, "dealine has exceeded, stoping server side operation")
		}
		if ctx.Err() == context.Canceled {
			log.Print("the user has canceled the request, stoping server side operation")
			return status.Error(codes.Canceled, "the user has canceled the request, stoping server side operation")
		}

	}

	log.Printf("the response was sent to client")

	return nil
}
{{< /code >}}

## Extend the GO client to support GetVendorProds RPC method

On the client side, we pass the method a context and a request. Instead of getting a response object back, we get back an instance of **ProdService_GetVendorProdsClient**. The client can use the ProdService_GetVendorProdsClient stream to read the server’s responses. We read the stream in an infinite loop untill **io.EOF** or other error is received. Finnaly we print out each product.

{{< code language="go" isCollapsed="false" >}}
func getprods(ctx context.Context, client api.ProdServiceClient, vendor string, prodType string) error {

	log.Printf("requesting all %s products from %s", prodType, vendor)

	if vendor == "" || prodType == "" {
		return fmt.Errorf("You need both, vendor and prodType args. Example command: $client oracle storage")
	}

	requestProd := api.ClientRequestProds{
		Vendor:      vendor,
		ProductType: prodType,
	}

	stream, err := client.GetVendorProds(ctx, &requestProd)
	if err != nil {
		if errStatus, ok := status.FromError(err); ok {
			return status.Errorf(errStatus.Code(), "error while calling client.GetVendorProds() method: %v ", errStatus.Message())
		}
		return fmt.Errorf("Could not get the stream of products : %v", err)
	}

	for {
		product, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			if errStatus, ok := status.FromError(err); ok {
				return status.Errorf(errStatus.Code(), "error while receiving the stream for client.GetVendorProds: %v ", errStatus.Message())
			}
			return fmt.Errorf("error while receiving the stream for client.GetVendorProds: %v", err)
		}
		fmt.Printf("Title: %s, Url: %s,  ShortUrl: %s\n", product.GetProduct().GetTitle(), product.GetProduct().GetUrl(), product.GetProduct().GetShortUrl())
	}

	return nil
}
{{< /code >}}

## Run the service

Bellow I'm running the service and show the results, the expected behaviour and the errors generated by incorrect arguments or deadline exceeded.

### The Happy Path

This is the expected behaviour. The logs from the API server and Backend Storage shows that everything went well.

{{< code language="bash" isCollapsed="false" >}}
# This is the Client
$ go run client.go getprods google compute
2020/01/13 17:29:16 requesting all compute products from google
Title: Compute Engine, Url: https://cloud.google.com/compute/,  ShortUrl: https://made-up-url.com/184e04
Title: App Engine, Url: https://cloud.google.com/appengine/,  ShortUrl: https://made-up-url.com/39d8a0
Title: Cloud Functions, Url: https://cloud.google.com/functions/,  ShortUrl: https://made-up-url.com/1510bb
Title: Cloud Run, Url: https://cloud.google.com/run/,  ShortUrl: https://made-up-url.com/3cb3a2
Title: GKE, Url: https://cloud.google.com/kubernetes-engine/,  ShortUrl: https://made-up-url.com/d61173

# This is the API Server
$ go run server.go 
2020/01/13 17:28:18 Serving gRPC on https://localhost:8080
2020/01/13 17:29:16 have received a request for -> compute <- product type from -> google <- vendor
2020/01/13 17:29:16 the response was sent to client

# This is the Backend Storage 
$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> compute <- product type from -> google <- vendor
INFO:root:a number of 5 products were sent to client
{{< /code >}}

### Incorrect arguments

This is with misspelled arguments. You can see "Exception calling application: Invalid ProductType: tools" propagated back to the client.

{{< code language="bash" isCollapsed="false" >}}
# This is the Client
$ go run client.go getprods google tools
2020/01/13 17:36:43 requesting all tools products from google
rpc error: code = Unknown desc = error while receiving the stream for client.GetVendorProds: error while calling client.GetProdsDetail() method: Exception calling application: Invalid ProductType: tools  
exit status 1

# This is the API Server
$ go run server.go 
2020/01/13 17:36:05 Serving gRPC on https://localhost:8080
2020/01/13 17:36:43 have received a request for -> tools <- product type from -> google <- vendor
2020/01/13 17:36:43 error while calling client.GetProdsDetail() method: Exception calling application: Invalid ProductType: tools

# This is the Backend Storage 
$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> tools <- product type from -> google <- vendor
ERROR:grpc._server:Exception calling application: Invalid ProductType: tools
Traceback (most recent call last):
  File "/home/rdan/python_env/env_grpc/lib/python3.7/site-packages/grpc/_server.py", line 435, in _call_behavior
    response_or_iterator = behavior(argument, context)
  File "storage.py", line 31, in GetProdsDetail
    prod_type_list = get_prods(vendor, product_type)
  File "storage.py", line 63, in get_prods
    raise Exception("Invalid ProductType: {}".format(product_type))
Exception: Invalid ProductType: tools
{{< /code >}}

### DeadLine Exceeded

We run again the request, but this time we simulate heavy processing on Backend and API Server, which generates the deadline exceeded error.
The context has to be configured on the client side **WithTimeout**:

{{< code language="go" isCollapsed="false" >}}
ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
{{< /code >}}

#### Heavy processing on the Backend Storage, 

If we add **time.sleep(5)** to the Backend Storage, the response is not received in time by the API Server and it generates the "context deadline exceeded" error.

{{< code language="bash" isCollapsed="false" >}}
# This is the Client
$ go run client.go getprods aws storage
2020/01/13 18:04:14 requesting all storage products from aws
rpc error: code = DeadlineExceeded desc = error while receiving the stream for client.GetVendorProds: context deadline exceeded 
exit status 1

# This is the API Server
$ go run server.go 
2020/01/13 18:03:56 Serving gRPC on https://localhost:8080
2020/01/13 18:04:14 have received a request for -> storage <- product type from -> aws <- vendor
2020/01/13 18:04:18 error while calling client.GetProdsDetail() method: context deadline exceeded

# This is the Backend Storage 
$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> storage <- product type from -> aws <- vendor
INFO:root:the connection to ipv4:127.0.0.1:46092 has dropped
{{< /code >}}

#### Heavy processing on the Api Server 

If we add **time.Sleep(1 * time.Second)** on the API Server after sending every instance of data in the stream, the "context deadline exceeded" is received after a while and the stream call is interrupted.

{{< code language="bash" isCollapsed="false" >}}
# This is the Client
$ go run client.go getprods oracle compute
2020/01/13 18:14:37 requesting all compute products from oracle
Title: Bare Metal Compute, Url: https://www.oracle.com/cloud/compute/bare-metal.html,  ShortUrl: https://made-up-url.com/feb785
Title: Container Engine for Kubernetes, Url: https://www.oracle.com/cloud/compute/container-engine-kubernetes.html,  ShortUrl: https://made-up-url.com/f6e3dd
Title: Virtual Machines, Url: https://www.oracle.com/cloud/compute/virtual-machines.html,  ShortUrl: https://made-up-url.com/5598d1
Title: Oracle Functions, Url: https://www.oracle.com/ro/cloud/cloud-native/functions/,  ShortUrl: https://made-up-url.com/8e9d1d
rpc error: code = DeadlineExceeded desc = error while receiving the stream for client.GetVendorProds: context deadline exceeded 
exit status 1

# This is the API Server
$ go run server.go 
2020/01/13 18:14:23 Serving gRPC on https://localhost:8080
2020/01/13 18:14:37 have received a request for -> compute <- product type from -> oracle <- vendor
2020/01/13 18:14:41 dealine has exceeded, stoping server side operation

# This is the Backend Storage 
$ python storage.py 
Listening on port 6000..
INFO:root:have received a request for -> compute <- product type from -> oracle <- vendor
INFO:root:a number of 4 products were sent to client
{{< /code >}}
