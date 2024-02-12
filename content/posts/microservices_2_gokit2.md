---
title: "Part 2 - cont: Microservices - Create the App with Go-kit"
date: 2019-06-28
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

This is the continuation of the previous blog post, where I explained the microservices architecture using GO Kit framework. I strongly recommend you to read the [previous post](https://dev-state.com/blog/microservices_2_gokit1/) as it gives you the insights into what I'll cover this post. I talked already about the two layers of the onion architecture: Service and Endpoints. In this post I’ll cover the Transport layer and put everything together in the main function.

In order to expose the service to the outside world, to be called, you can use one of the GO-Kit supported transports. It provides support for HTTP, gRPC, NATs and Trift. For the Stats service I’m using gRPC and Frontend Service expose JSON over HTTP.  Frontend service also act as a grpc client for the Stats service.

Before diving into the implementation details, let’s remind you what is Protocol buffer and gRPC. 

### Protocol Buffers:

Protocol buffers are a flexible, efficient, automated mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. It is more efficient than XML and JSON as the transported message is encoded in binary format while transferred over wire.

The [Protocol buffers](https://developers.google.com/protocol-buffers/) site has a ton of documentation.

I define a  .proto file which contain the messages and services. For Stats service it looks like this:

{{< code language="go" isCollapsed="false" >}}
syntax = "proto3";

package pb;

// -------------- Stats Service ------------------

service StatsService {
    rpc ListTable(TableRequest) returns (TableReply) {}
    rpc ListTeamPlayers(TeamRequest) returns (TeamReply) {}
    rpc ListPositionPlayers(PositionRequest) returns (PositionReply) {}
}

message TableRequest {
    string tableName = 1;
}

message Table {
    string teamName = 1;
    int32 teamPlayed = 2;
    int32 teamWon = 3;
    int32 teamDrawn = 4;
    int32 teamLost = 5;
    int32 teamGF = 6;
    int32 teamGA = 7;
    int32 teamGD = 8;
    int32 teamPoints = 9;
    int32 teamCapital = 10;
}

message TableReply {
    repeated Table teams = 1;
    string err = 2;

}

message TeamRequest {
    string teamName = 1;
}

message Player {
    string name = 1;
    string team = 2;
    string nationality = 3;
    string position = 4;
    int32 appearences = 5;
    int32 goals = 6;
    int32 assists = 7;
    int32 passes = 8;
    int32 interceptions = 9;
    int32 tackles = 10;
    int32 fouls = 11;
}

message TeamReply {
    repeated Player players = 1;
    string err = 2;
}

message PositionRequest {
    string position = 1;
}

message PositionReply {
    repeated Player players = 1;
    string err = 2;
}
{{< /code >}}

Protocol buffers can generate code  in a wider range of programming languages like Java, C++, Python, Ruby, JavaScript,  in addition you can generate proto3 code for Go using the latest Go protoc plugin.

### GRPC:

gRPC (gRPC Remote Procedure Calls) is an open source remote procedure call (RPC) system initially developed at Google. It uses HTTP/2 for transport, Protocol Buffers as the Interface Description Language, and provides features such as authentication, bidirectional streaming and flow control. It generates cross-platform client and server bindings for many languages.  
In gRPC a client application can directly call methods on a server application on a different machine as if it was a local object, making it easier for you to create distributed applications and services. As in many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, the client has a stub (referred to as just a client in some languages) that provides the same methods as the server.

The [GRPC](https://grpc.io/) site has a ton of documentation as well.

{{< image src="/img/2019/grpc.svg" style="border-radius: 8px;" >}}

To make it work with GO, you have to install the grpc package and protoc plugin for go:

{{< code language="go" isCollapsed="false" >}}
 go get -u google.golang.org/grpc
 go get -u github.com/golang/protobuf/protoc-gen-go
{{< /code >}}

Install the protoc compiler from here https://github.com/protocolbuffers/protobuf/releases, that is used to generate gRPC service code. To generate the code you have to run:

{{< code language="bash" isCollapsed="false" >}}
protoc --go_out=plugins=grpc:. pb/filename.proto
{{< /code >}}

The filename.pb.go file is created as a result of running the above command, which generates of bunch of go code, including the structs for request and reply as defined above and the StatsServiceClient ,which is the client API for StatsService  and StatsServiceServer which is the server API for StatsService and has to be implemented on the server side.

### Application:

The Go kit package github.com/go-kit/kit/transport/grpc provides the support for serving services with GRPC transport. The function NewServer of transport/grpc package constructs a new server, which implements wraps the provided endpoint and implements the Handler interface, has the following form:

{{< code language="go" isCollapsed="false" >}}
func NewServer(
    e endpoint.Endpoint,
    dec DecodeRequestFunc,
    enc EncodeResponseFunc,
    options ...ServerOption,
) *Server
{{< /code >}}

NewGRPCServer takes a set of endpoints and makes them callable by gRPC clients. This means the "outside" of the server needs to speak protobufs to clients (e.g. pb.TableReply) and the "inside" needs to speak whatever the endpoints expect (e.g. endpoints.TableReply).

{{< code language="go" isCollapsed="false" >}}
type gRPCServer struct {
	listTable           gt.Handler
	listTeamPlayers     gt.Handler
	listPositionPlayers gt.Handler
}

// NewGRPCServer makes a set of endpoints available as a gRPC StatsServiceServer.
func NewGRPCServer(statsEndpoints endpoints.Endpoints, logger log.Logger) pb.StatsServiceServer {
	return &gRPCServer{
		listTable: gt.NewServer(
			statsEndpoints.ListTableEndpoint,
			decodeListTableRequest,
			encodeListTableResponse,
		),
		listTeamPlayers: gt.NewServer(
			statsEndpoints.ListTeamPlayersEndpoint,
			decodeListTeamPlayers,
			encodeListTeamPlayers,
		),
		listPositionPlayers: gt.NewServer(
			statsEndpoints.ListPositionPlayersEndpoint,
			decodeListPositionPlayers,
			encodeListPositionPlayers,
		),
	}
}
{{< /code >}}

Then we have to implement the  StatsServiceServer interface by creating the below methods:

{{< code language="go" isCollapsed="false" >}}
func (s *gRPCServer) ListTable(ctx context.Context, req *pb.TableRequest) (*pb.TableReply, error) {
	_, resp, err := s.listTable.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return resp.(*pb.TableReply), nil
}

func (s *gRPCServer) ListTeamPlayers(ctx context.Context, req *pb.TeamRequest) (*pb.TeamReply, error) {
	_, resp, err := s.listTeamPlayers.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return resp.(*pb.TeamReply), nil
}

func (s *gRPCServer) ListPositionPlayers(ctx context.Context, req *pb.PositionRequest) (*pb.PositionReply, error) {
	_, resp, err := s.listPositionPlayers.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PositionReply), nil
}
{{< /code >}}

**DecodeRequestFunc** extracts a user-domain request object from a gRPC request. In my case the decodeListTableRequest  takes the pb.TableRequest input from a client, and return a endpoints.TableRequest output for further processing by the
microservice.   
**EncodeResponseFunc** encodes the passed response object to the gRPC response message.
In my case the encodeListTableResponse function is to go the other direction: take the endpoints.TableReply input from the microservice, and return a pb.TableReply output to return to the client. 

{{< code language="go" isCollapsed="false" >}}
unc decodeListTableRequest(_ context.Context, request interface{}) (interface{}, error) {
	req := request.(*pb.TableRequest)
	return endpoints.TableRequest{League: req.TableName}, nil
}

func encodeListTableResponse(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(endpoints.TableReply)

	teams := make([]*pb.Table, len(resp.Teams))
	for i := range resp.Teams {
		teams[i] = &pb.Table{
			TeamName:    resp.Teams[i].TeamName,
			TeamPlayed:  resp.Teams[i].TeamPlayed,
			TeamWon:     resp.Teams[i].TeamWon,
			TeamDrawn:   resp.Teams[i].TeamDrawn,
			TeamLost:    resp.Teams[i].TeamLost,
			TeamGF:      resp.Teams[i].TeamGF,
			TeamGA:      resp.Teams[i].TeamGA,
			TeamGD:      resp.Teams[i].TeamGD,
			TeamPoints:  resp.Teams[i].TeamPoints,
			TeamCapital: resp.Teams[i].TeamCapital,
		}
	}

	return &pb.TableReply{Teams: teams, Err: err2str(resp.Err)}, nil
}

func decodeListTeamPlayers(_ context.Context, request interface{}) (interface{}, error) {
	req := request.(*pb.TeamRequest)
	return endpoints.TeamRequest{TeamName: req.TeamName}, nil
}

func encodeListTeamPlayers(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(endpoints.TeamReply)

	players := make([]*pb.Player, len(resp.Players))
	for i := range resp.Players {
		players[i] = &pb.Player{
			Name:          resp.Players[i].Name,
			Team:          resp.Players[i].Team,
			Nationality:   resp.Players[i].Nationality,
			Position:      resp.Players[i].Position,
			Appearences:   resp.Players[i].Appearences,
			Goals:         resp.Players[i].Goals,
			Assists:       resp.Players[i].Assists,
			Passes:        resp.Players[i].Passes,
			Interceptions: resp.Players[i].Interceptions,
			Tackles:       resp.Players[i].Tackles,
			Fouls:         resp.Players[i].Fouls,
		}
	}

	return &pb.TeamReply{Players: players, Err: err2str(resp.Err)}, nil
}

func decodeListPositionPlayers(_ context.Context, request interface{}) (interface{}, error) {
	req := request.(*pb.PositionRequest)
	return endpoints.PositionRequest{Position: req.Position}, nil
}

func encodeListPositionPlayers(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(endpoints.PositionReply)

	players := make([]*pb.Player, len(resp.Players))
	for i := range resp.Players {
		players[i] = &pb.Player{
			Name:          resp.Players[i].Name,
			Team:          resp.Players[i].Team,
			Nationality:   resp.Players[i].Nationality,
			Position:      resp.Players[i].Position,
			Appearences:   resp.Players[i].Appearences,
			Goals:         resp.Players[i].Goals,
			Assists:       resp.Players[i].Assists,
			Passes:        resp.Players[i].Passes,
			Interceptions: resp.Players[i].Interceptions,
			Tackles:       resp.Players[i].Tackles,
			Fouls:         resp.Players[i].Fouls,
		}
	}

	return &pb.PositionReply{Players: players, Err: err2str(resp.Err)}, nil
}

// Helper function is required to translate Go error types to strings,
// which is the type we use in our IDLs to represent errors.

func err2str(err error) string {
	if err == nil {
		return ""
	}
	return err.Error()
}
{{< /code >}}

The GRPC Stub is implemented on the Frontend service.

Last thing before wrapping up is to create the main.go which instantiate the service, endpoints and grpc server.  
Define the log and create the firestore client:

{{< code language="go" isCollapsed="false" >}}
var grpcAddr = ":8081"

var logger log.Logger
logger = log.NewLogfmtLogger(os.Stderr)
logger = log.With(logger, "ts", log.DefaultTimestampUTC)
logger = log.With(logger, "caller", log.DefaultCaller)

level.Info(logger).Log("msg", "service started")
defer level.Info(logger).Log("msg", "service ended")

// add database with credentials to run locally
ctx := context.Background()
var firestoreClient *firestore.Client
sa := option.WithCredentialsFile("../keys/apps-microservices-68b9b8c44847.json")
firestoreClient, err := firestore.NewClient(ctx, "apps-microservices", sa)
if err != nil {
	logger.Log("database", "firestore", "during", "ClientCreation", "err", err)
	os.Exit(1)
}

defer firestoreClient.Close()
{{< /code >}}

Build the layers of the service "onion" from the inside out. First, the business logic service; then the set of endpoints that wrap the service and finally, a series of concrete transport adapters.

{{< code language="go" isCollapsed="false" >}}
addservice := service.NewStatsService(firestoreClient, logger)
addendpoints := endpoints.MakeStatsEndpoints(addservice)
grpcServer := transport.NewGRPCServer(addendpoints, logger)
{{< /code >}}

Runs the gRPC server to handle client calls and gracefully close the server connection if SIGINT is received via terminal or SIGTERM.

{{< code language="go" isCollapsed="false" >}}
errs := make(chan error)
go func() {
	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	errs <- fmt.Errorf("%s", <-c)
}()

grpcListener, err := net.Listen("tcp", grpcAddr)
if err != nil {
	logger.Log("transport", "gRPC", "during", "Listen", "err", err)
	os.Exit(1)
}

go func() {
	level.Info(logger).Log("transport", "GRPC", "addr", grpcAddr)
	baseServer := grpc.NewServer()
	pb.RegisterStatsServiceServer(baseServer, grpcServer)
	baseServer.Serve(grpcListener)
}()

level.Error(logger).Log("exit", <-errs)
{{< /code >}}

The Frontend Service invoke Stats Service via GRPC and perform some business logic to display the requested information to the user. It is worth to notice that Frontend expose HTTP transport to the user and use gorilla mux under the hood. The http endpoint has this form, which is similar to grpc one:

{{< code language="go" isCollapsed="false" >}}
func NewServer(
    e endpoint.Endpoint,
    dec DecodeRequestFunc,
    enc EncodeResponseFunc,
    options ...ServerOption,
) *Server
{{< /code >}}

Server wraps an endpoint and implements http.Handler, a code example from Frontend transport.go:

{{< code language="go" isCollapsed="false" >}}
r.Methods("GET").Path("/bestplayers/{team}").Handler(kithttp.NewServer(
	siteEndpoints.GetTeamBestPlayersEndpoint,
	decodeGetTeamRequest,
	encodeResponse,
	options...,
))
{{< /code >}}

### Try the application:

Build or run both services (Stats and Frontend) and run below commands. I'm using a linux terminal. List best player of a selected team for each position.

{{< code language="bash" isCollapsed="false" >}}
curl http://localhost:8080/bestplayers/Manchester%20City
{{< /code >}}

the result:

{{< code language="json" isCollapsed="false" >}}
{"Players":[{"name":"Aymeric Laporte","team":"Manchester City","nationality":"France","position":"Defender","appearences":35,"goals":3,"assists":3,"passes":2998,"interceptions":39,"tackles":42,"fouls":21},
{"name":"Fernandinho","team":"Manchester City","nationality":"Brazil","position":"Midfielder","appearences":29,"goals":1,"assists":3,"passes":2050,"interceptions":41,"tackles":57,"fouls":40},
{"name":"Sergio Aguero","team":"Manchester City","nationality":"Argentina","position":"Forward","appearences":33,"goals":21,"assists":8,"passes":771,"interceptions":9,"tackles":17,"fouls":21}],"Err":null}
{{< /code >}}

-----------------

List the stats of all the teams from the database:

{{< code language="bash" isCollapsed="false" >}}
curl --header "Content-Type: application/json" --request GET --data '{"league":"League"}' http://localhost:8080/table
{{< /code >}}

the result:

{{< code language="json" isCollapsed="false" >}}
{"teams":[{"teamName":"Manchester City","teamPlayed":38,"teamWon":32,"teamDrawn":2,"teamLost":4,"teamGF":95,"teamGA":23,"teamGD":72,"teamPoints":98,"teamCapital":300},
{"teamName":"Liverpool","teamPlayed":38,"teamWon":30,"teamDrawn":7,"teamLost":1,"teamGF":89,"teamGA":22,"teamGD":67,"teamPoints":97,"teamCapital":250},
{"teamName":"Chelsea","teamPlayed":38,"teamWon":21,"teamDrawn":9,"teamLost":8,"teamGF":63,"teamGA":39,"teamGD":24,"teamPoints":72,"teamCapital":200},
{"teamName":"Tottenham Hotspur","teamPlayed":38,"teamWon":23,"teamDrawn":2,"teamLost":13,"teamGF":67,"teamGA":39,"teamGD":28,"teamPoints":71,"teamCapital":150}],"err":null}
{{< /code >}}

-----------------

List best league players on a selected position: 

{{< code language="bash" isCollapsed="false" >}}
curl --header "Content-Type: application/json" --request GET --data '{"position":"Defender"}' http://localhost:8080/bestposition
{{< /code >}}

the result:

{{< code language="json" isCollapsed="false" >}}
{"Players":[{"name":"Cesar Azpilicueta","team":"Chelsea","nationality":"Spain","position":"Defender","appearences":38,"goals":1,"assists":5,"passes":2483,"interceptions":43,"tackles":105,"fouls":39},
{"name":"Marcos Alonso","team":"Chelsea","nationality":"Spain","position":"Defender","appearences":31,"goals":2,"assists":4,"passes":1881,"interceptions":39,"tackles":74,"fouls":28},
{"name":"Andrew Robertson","team":"Liverpool","nationality":"Scotland","position":"Defender","appearences":36,"goals":0,"assists":11,"passes":2396,"interceptions":30,"tackles":80,"fouls":18}],"Err":null}
{{< /code >}}

-----------------

The logs from the Frontend service:

{{< code language="bash" isCollapsed="false" >}}
level=info ts=2019-07-05T10:56:55.96584837Z caller=main.go:42 msg="service started"
level=info ts=2019-07-05T10:56:55.966003598Z caller=main.go:71 transport=HTTP addr=:8080
ts=2019-07-05T10:58:53.003957525Z caller=middleware.go:33 method=GetTeamBestPlayers teamName="Manchester City" err=null
ts=2019-07-05T10:59:35.89750468Z caller=middleware.go:26 method=GetTable league=League err=null
ts=2019-07-05T11:00:06.389554192Z caller=middleware.go:40 method=GetPositionBestPlayers position=Defender err=null
^Clevel=error ts=2019-07-05T11:00:31.596482753Z caller=main.go:79 exit=interrupt
level=info ts=2019-07-05T11:00:31.596604227Z caller=main.go:80 msg="service ended"
{{< /code >}}

The logs from the Stats service:

{{< code language="bash" isCollapsed="false" >}}
level=info ts=2019-07-05T10:56:50.505200223Z caller=main.go:32 msg="service started"
level=info ts=2019-07-05T10:56:50.505544193Z caller=main.go:69 transport=GRPC addr=:8081
ts=2019-07-05T10:58:53.003217568Z caller=middleware.go:33 method=ListTeamPlayers teamName="Manchester City" err=null
ts=2019-07-05T10:59:35.896928982Z caller=middleware.go:26 method=Listable league=League err=null
ts=2019-07-05T11:00:06.388854565Z caller=middleware.go:40 method=ListPostionPlayers position=Defender err=null
^Clevel=error ts=2019-07-05T11:00:29.646112475Z caller=main.go:75 exit=interrupt
level=info ts=2019-07-05T11:00:29.646230416Z caller=main.go:76 msg="service ended"
{{< /code >}}

### Conclusion:

In the [first section](https://dev-state.com/posts/microservices_1_setup/) I showed you how to ingest data at scale using the Serverless Infrastructure. In the last 2 blog posts, part of the [section 2](https://dev-state.com/posts/microservices_2_gokit1/), I created two services which communicate between each other using gRPC and expose HTTP REST endpoint externally.  
The complete code is available on [Github](https://github.com/danrusei/microservices_project/tree/master/section_2).  
The Go Kit framework helped me to accomplish this goal, as it provides good structure of the code and enforce the separation of concerns (onion software architecture), which I believe it makes the code extensible and readable.  
There are only two services, in the next section I'll add a couple more. I'll cover how to package these services and deploy on kubernetes platform, how to monitor and instrument the services. 
