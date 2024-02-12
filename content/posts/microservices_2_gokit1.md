---
title: "Part 2: Microservices - Create the App with Go-kit"
date: 2019-06-24
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

This is the second blog post of the series, where I dive into the process details and the framework I used to create the toy project. It is made up of several distinct microservices.
There are a number of microservice frameworks in the wild but the most notable ones for GO are [Go Micro](https://micro.mu/), [Go-Kit](https://gokit.io/), [Gizmo](https://github.com/NYTimes/gizmo). 

Go-Kit is the one which has prompted my interest, I’m explaining below why. Asa starting point I’m creating a small application, formed by two microservices. The Frontend is a  REST endpoint which exposes the HTTP GET methods and return the message to user within JSON format. This function communicate with the backend service Stats to retrieve data via GRPC/protobuf.

### Application Description:

First, why Microservices? It is one of the most popular buzz-word in the field of software architecture. You can search on google and you’ll find a large number of people explaining the microservices and the benefits of using them over monolithic architecture.

A very comprehensive definition can be found on Wikipedia:  
"Microservices are a software development technique, a variant of the service-oriented architecture (SOA) architectural style that structures an application as a collection of loosely coupled services.   
In a microservices architecture, services are fine-grained and the protocols are lightweight. The benefit of decomposing an application into different smaller services is that it improves modularity. This makes the application easier to understand, develop, test, and become more resilient to architecture erosion.   
It parallelizes development by enabling small autonomous teams to develop, deploy and scale their respective services independently. It also allows the architecture of an individual service to emerge through continuous refactoring. Microservice-based architectures enable continuous delivery and deployment."  

Though, there is a cost on moving to microservices. Inter-service calls over a network have a higher cost in terms of network latency. Testing, monitoring, troubleshooting and deployment aren’t easy comparing with monolithic service architecture. This is where microservices frameworks  like Go-Kit and platforms like Kubernetes with ISTIO becomes handy. In the interest of simplicity I’m starting with just two services and I’ll expand with another two later on. That enables me to better explain the concepts and  to progressively introduce new features. 

The complete code is available on [GITHUB](https://github.com/danrusei/microservices_project/tree/master/section_2)

### GO KIT:

Go kit a lightly opinionated microservice toolkit, made up by  a collection of Go  packages that help you build robust, reliable, maintainable microservices. Go kit microservices are modeled like an onion, with many layers. The layers can be grouped into three domains: Service, Endpoint and Transport. The  Onion Architecture which was introduced by Jeffrey Palermo to provide a better way to build applications in perspective of better testability, maintainability, and dependability it helps you embrace a solid design principles.

{{< image src="/img/2019/onion.png" style="border-radius: 8px;" >}}

The Go Kit layers provides a good separation of concerns. Each service method converts as an endpoint by using an adapter and it’s exposed by using concrete transports. Therefore can be attached multiple transports to the same service.

Service layer:
The service domain is where everything is based on your specific service definition, and where all of the business logic is implemented.

Endpoint layer:
The Go kit primary messaging pattern is RPC, and an endpoint represents a single RPC method. Each method of our service needs to be wrapped in an endpoint. 

Transport layer
The services often communicate to each other using concrete transports like HTTP or gRPC, or using a pub/sub systems like NATS. Go Kit support  HTTP, gRPC, NATS, AMQP and JSON RPC. The business logic doesn’t have to know about concrete transports.

Another powerful mechanism in GO KIt, is that it tries to enforce strict separation of concerns through the use of the **Middleware** pattern. Middlewares can wrap endpoints or services to add functionality, such as logging, rate limiting, load balancing, or distributed tracing. It’s common to chain multiple middlewares around an endpoint or service.
It supports a vast number of functionalities: Circuitbreaker, Rate limit, Service Discovery ( consul ), Observability ( Prometheus ) ,  Tracing ( zipkin and opencensus ) and many more which you may find useful. 
It is out of the scope of this blog post to cover all these functionalities, instead as the services are created using Go-Kit framework, all of these are pluggable. In the next blog post I’ll explain the benefits of using the sidecar model, for Kubernetes deployed services, instead of embedding them into the code.

### Application:

To start with, I’m creating two services, Frontend and Stats. They are modeled following the same architecture. As I’m explaining the concepts for Stats Service, these applies to Frontend Service as well. The Frontend Service expose the REST endpoint and communicate over grpc to internal Stats Service.

In the **model.go** I define the structs, which holds Player and league’s Table data. They will be used throughout the entire program.  
The league’s Table struct contains the overall performance of the team, like how many games were played, won, lost and drawn. It includes the number of goals were scored, against and the difference. The last 2 fields are the Points achieved at the end of the season and the Capital which will be used by Transfer Service latter on.  
The Player struct contains the player detals and some statistics from the end of the season, which are used by the service to determine the Best Forward, Defender and Midfielder.     

{{< code language="go" isCollapsed="false" >}}
//Table struct holds League table data
type Table struct {
	TeamName    string `firestore:"Name"`
	TeamPlayed  int32  `firestore:"Played"`
	TeamWon     int32  `firestore:"Won"`
	TeamDrawn   int32  `firestore:"Drawn"`
	TeamLost    int32  `firestore:"Lost"`
	TeamGF      int32  `firestore:"GF"`
	TeamGA      int32  `firestore:"GA"`
	TeamGD      int32  `firestore:"GD"`
	TeamPoints  int32  `firestore:"Points"`
	TeamCapital int32  `firestore:"Capital"`
}

//Player struct holds player data
type Player struct {
	Name          string `firestore:"player"`
	Team          string `firestore:"team"`
	Nationality   string `firestore:"nationality"`
	Position      string `firestore:"position"`
	Appearences   int32  `firestore:"appearences"`
	Goals         int32  `firestore:"goals"`
	Assists       int32  `firestore:"assists"`
	Passes        int32  `firestore:"passes"`
	Interceptions int32  `firestore:"Interceptions"`
	Tackles       int32  `firestore:"Tackles"`
	Fouls         int32  `firestore:"Fouls"`
	//	Price         int32  `firestore:"Price" json:"price"`
}
{{< /code >}}

Create the **service.go** file, which contains the business logic. The business logic is implemented in services and modelled as interface.

{{< code language="go" isCollapsed="false" >}}
//StatsService describe the Stats service
type StatsService interface {
	ListTable(ctx context.Context, league string) ([]Table, error)
	ListTeamPlayers(ctx context.Context, teamName string) ([]Player, error)
	ListPositionPlayers(ctx context.Context, position string) ([]Player, error)
}
{{< /code >}}

The **basicService** struct implement the StatsService interface. I use Firestore as a persistent storage.  
The instantiated firestore.Client is injected in  the the basicService struct using the NewStatsService function. Also the service requests are logged using a middleware, which I will detail below.    
ListTable invokes **DataTo** method, which uses the document's fields to populate the Table struct. In a similar manner ListTeamPlayers will get all players part of the same team from database and ListPositionPlayer will get all the players playing the same position from the database.  
Checkout [Part 1](http://localhost:1313/blog/microservices_1_setup/) of these series for references on Firestore and how to work with it.

{{< code language="go" isCollapsed="false" >}}
// NewStatsService returns a basic StatsService with all of the expected middlewares wired in.
func NewStatsService(client *firestore.Client, logger log.Logger) StatsService {
	var svc StatsService
	svc = NewBasicService(client)
	svc = LoggingMiddleware(logger)(svc)
	return svc
}

// NewBasicService returns a naive, stateless implementation of StatsService.
func NewBasicService(client *firestore.Client) StatsService {
	return &basicService{
		dbClient: client,
	}
}

type basicService struct {
	dbClient *firestore.Client
}

func (s *basicService) ListTable(ctx context.Context, league string) ([]Table, error) {

	var teamTable Table
	var leagueTable []Table

	leagueDocs := s.dbClient.Collection(league)
	q := leagueDocs.OrderBy("Points", firestore.Desc)
	iter := q.Documents(ctx)
	defer iter.Stop()
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, ErrIterate
		}
		if err := doc.DataTo(&teamTable); err != nil {
			return nil, ErrExtractDataToStruct
		}
		leagueTable = append(leagueTable, teamTable)
	}

	return leagueTable, nil
}

func (s *basicService) ListTeamPlayers(ctx context.Context, teamName string) ([]Player, error) {

	var singlePlayer Player
	var teamPlayers []Player

	teamsDocs := s.dbClient.Collection("Teams")
	q := teamsDocs.Where("team", "==", teamName)
	//.OrderBy("player", firestore.Desc)
	iter := q.Documents(ctx)
	//iter := s.dbClient.Collection("Teams").Documents(ctx)
	defer iter.Stop()
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, ErrIterate
		}
		if err := doc.DataTo(&singlePlayer); err != nil {
			return nil, ErrExtractDataToStruct
		}
		teamPlayers = append(teamPlayers, singlePlayer)
	}

	return teamPlayers, nil
}

func (s *basicService) ListPositionPlayers(ctx context.Context, position string) ([]Player, error) {

	var singlePlayer Player
	var teamPlayers []Player

	teamsDocs := s.dbClient.Collection("Teams")
	q := teamsDocs.Where("position", "==", position)
	//.OrderBy("team", firestore.Desc)
	iter := q.Documents(ctx)
	defer iter.Stop()
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, ErrIterate
		}
		if err := doc.DataTo(&singlePlayer); err != nil {
			return nil, ErrExtractDataToStruct
		}
		teamPlayers = append(teamPlayers, singlePlayer)
	}

	return teamPlayers, nil
}
{{< /code >}}

I mentioned earlier that the Middlewares are very powerful in Go kit. I’m using below the service middleware to log the service calls, including parameters that are passed in.  
I create a separate file called **middleware.go**.

{{< code language="go" isCollapsed="false" >}}
// Middleware describes a service (as opposed to endpoint) middleware.
type Middleware func(StatsService) StatsService

// LoggingMiddleware takes a logger as a dependency and returns a ServiceMiddleware.
func LoggingMiddleware(logger log.Logger) Middleware {
	return func(next StatsService) StatsService {
		return loggingMiddleware{logger, next}
	}
}

type loggingMiddleware struct {
	logger log.Logger
	next   StatsService
}

func (mw loggingMiddleware) ListTable(ctx context.Context, league string) (t []Table, err error) {
	defer func() {
		mw.logger.Log("method", "Listable", "league", league, "err", err)
	}()
	return mw.next.ListTable(ctx, league)
}

func (mw loggingMiddleware) ListTeamPlayers(ctx context.Context, teamName string) (p []Player, err error) {
	defer func() {
		mw.logger.Log("method", "ListTeamPlayers", "teamName", teamName, "err", err)
	}()
	return mw.next.ListTeamPlayers(ctx, teamName)
}

func (mw loggingMiddleware) ListPositionPlayers(ctx context.Context, position string) (p []Player, err error) {
	defer func() {
		mw.logger.Log("method", "ListPostionPlayers", "position", position, "err", err)
	}()
	return mw.next.ListPositionPlayers(ctx, position)
}
{{< /code >}}

In Go kit, the primary messaging pattern is RPC. So, every method in the interface will be modeled as a remote procedure call. For each method, **request and response** structs should be defined, which are used for RPC endpoints.

{{< code language="go" isCollapsed="false" >}}
//TableRequest holds the request params for ListTables
type TableRequest struct {
	League string
}

//TableReply holds the response params for ListTables
type TableReply struct {
	Teams []service.Table 
	Err   error           
}

//TeamRequest holds the request params for ListTeamPLayers
type TeamRequest struct {
	TeamName string
}

//TeamReply holds the response params for ListTeamPlayers
type TeamReply struct {
	Players []service.Player
	Err     error
}

//PositionRequest holds the request paramas for ListPositionPlayers
type PositionRequest struct {
	Position string
}

//PositionReply holds the response paramas for ListPositionPlayers
type PositionReply struct {
	Players []service.Player
	Err     error
}
{{< /code >}}

The services are exposed as RPC endpoints using a Go kit abstraction called endpoint. Any interaction with the service will be through the Endpoint.   
An endpoint is defined as follows:

{{< code language="go" isCollapsed="false" >}}
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
{{< /code >}}

Below, I’m  writing the adapters to convert each of our service’s methods into an endpoint. Each adapter takes a StatsService, and returns an endpoint that corresponds to one of the methods.

{{< code language="go" isCollapsed="false" >}}
//Endpoints holds all Stats Service enpoints
type Endpoints struct {
	ListTableEndpoint           endpoint.Endpoint
	ListTeamPlayersEndpoint     endpoint.Endpoint
	ListPositionPlayersEndpoint endpoint.Endpoint
}

//MakeStatsEndpoints initialize all service Endpoints
func MakeStatsEndpoints(s service.StatsService) Endpoints {
	return Endpoints{
		ListTableEndpoint:           makeListTableEndpoint(s),
		ListTeamPlayersEndpoint:     makeListTeamPLayersEndpoint(s),
		ListPositionPlayersEndpoint: makeListPositionPlayersEnpoint(s),
	}
}

func makeListTableEndpoint(s service.StatsService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(TableRequest)
		table, err := s.ListTable(ctx, req.League)
		return TableReply{Teams: table, Err: err}, nil
	}
}

func makeListTeamPLayersEndpoint(s service.StatsService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(TeamRequest)
		teamPlayers, err := s.ListTeamPlayers(ctx, req.TeamName)
		return TeamReply{Players: teamPlayers, Err: err}, nil
	}
}

func makeListPositionPlayersEnpoint(s service.StatsService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(PositionRequest)
		positionPlayers, err := s.ListPositionPlayers(ctx, req.Position)
		return PositionReply{Players: positionPlayers, Err: err}, nil
	}

}
{{< /code >}}

The application is not yet finalized as we have to add the Transport layer and to call all of them from within the main function. I'm covering these in the next blog post which is still part of the Section 2 of the series.  
