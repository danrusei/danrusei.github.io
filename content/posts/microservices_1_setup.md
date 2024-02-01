---
title: "Part 1: Microservices - Ingesting data"
date: 2019-06-15
draft: false
tags: ["Microservices", "Firestore"]
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

I’m starting a series of 5 blog posts where I document the path I followed to create a small project which is made up of several components (microservices) deployed over a  managed kubernetes infrastructure.   
In this first blog post I create a simple serverless data ingest pipeline, the next 2 blog posts are in depth description of the microservices architecture and the last 2 blog posts are about deploying the application in a Kubernetes environment and the mechanism to manage and monitor the services.

### Application description:

The application is made up of a  set of services which communicate between each other using GRPC /Protocol Buffers and a RESTful JSON API that is exposed to the outside world.  
The underlying data is a subset of statistics of the top premier league clubs available on https://www.premierleague.com/ site. The model is extensible to all clubs, to more leagues or even other sports, but as this is just a sample application I’m using the data from the top 4 ranked clubs of 2018/19 football season . 

### Data Ingestion:

I could have inserted the data manually in some sort of databases but I thought it would be more fun to use an automated ingestion mechanism, which can be tweaked and adapted for different use cases. I’m using Google Cloud platform to host the application and the dataset. The data ingestion mechanism is using a Cloud Function, which is triggered by the Storage when a new csv file is uploaded in the specified bucket. The Function reads the file and insert the data in a Firestore NoSQL database.

The complete code is available on [GITHUB](https://github.com/danrusei/microservices_project/tree/master/section_1), and the sample csv files can be found in the data directory. The cvs files contain the following fields: Team, Player, Nationality, Position, Appearances, Goals, Assists, Passes, Interceptions, Tackles, Fouls and estimated player Price. Now let me explain how the script works and the architecture design.

### Cloud Functions:

Google Cloud Functions is a serverless execution environment for building and connecting cloud services. Developers write simple, single-purpose functions that are attached to events triggered by cloud services or other applications. As this is serverless, there is no need to provision any infrastructure or to worry about managing any servers. The service abstracts and automates the underlying infrastructure, scale it up and down in response to function demand. As per time I’m writing this blog post the Cloud Functions supports  Node.js (Node.js 6, 8 or 10), Python 3 (Python 3.7) or Go (Go 1.11) as runtime environments. 

{{< image src="/img/2019/how-it-works.svg" style="border-radius: 8px;" >}}

Cloud events are things that happen in your cloud environment, like a file upload in the storage,  message published in Pub/Sub, changes to data in database. You can choose to respond to events via a Trigger.  A trigger is a declaration that you are interested in a certain event or set of events. When an event triggers the execution of your Cloud Function, data associated with the event is passed via the function's parameters.
The type of event determines the parameters passed to your function. In the Go runtime, functions take the following parameters:

* **HTTP Functions** -- the function is passed the net/http parameters (http.ResponseWriter, *http.Request). Use the ResponseWriter parameter to send a response.
* **Background Functions** -- the function is passed the parameters (context.Context, Event).

In current project the Cloud Function is triggered by Cloud Storage event. Every time an object is created in a Cloud Storage bucket the function is invoked. Background functions are passed arguments holding data associated with the event that triggered the function's execution.

* **ctx**  -- a context.Context value which carries metadata about the event.
* **Event** -- a struct, whose type is defined by developer, which the event payload will be unmarshaled into using json.Unmarshal() . 
The event payload depends on the trigger for which the function was registered.

{{< code language="go" isCollapsed="false" >}}
//GCSEvent is the payload of a GCS event.
type GCSEvent struct {
	Bucket         string    `json:"bucket"`
	Name           string    `json:"name"`
	Metageneration string    `json:"metageneration"`
	ResourceState  string    `json:"resourceState"`
	TimeCreated    time.Time `json:"timeCreated"`
	Updated        time.Time `json:"updated"`
}
{{< /code >}}

The event metadata and event payload can be logged and used to start the main workflow. 
Here we log the event details and execute getFileFromGCS function.

{{< code language="go" isCollapsed="false" >}}
// ToFirestore reads GCS file and upload contet to Firestore
func ToFirestore(ctx context.Context, e GCSEvent) error {
	meta, err := metadata.FromContext(ctx)
	if err != nil {
		return fmt.Errorf("metadata.FromContext: %v", err)
	}
	log.Printf("Event ID: %v\n", meta.EventID)
	log.Printf("Event type: %v\n", meta.EventType)
	log.Printf("Bucket: %v\n", e.Bucket)
	log.Printf("File: %v\n", e.Name)
	log.Printf("Metageneration: %v\n", e.Metageneration)
	log.Printf("Created: %v\n", e.TimeCreated)
	log.Printf("Updated: %v\n", e.Updated)

	//copy file from gs bucket
	//read csv file and populate Team struct
	//insert data in Firestore

	teams, err := getFileFromGCS(e.Bucket, e.Name)
	if err != nil {
		log.Printf("could not construct the struct : %v", err)
	}

	err = insertInFirestore(teams)
	if err != nil {
		log.Printf("could not create the Team Document in Firestore: %v: ", err)
	}
	//teamsJSON, _ := json.Marshal(teams)
	//fmt.Println(string(teamsJSON))

	return nil
}
{{< /code >}}

**getFileFromGCS** function is invoked, that takes as arguments bucket name and the filename. The provided informations are required to read the file from Cloud Storage and marshal to a defined go struct.
You need cloud.google.com/go/storage package installed and the client to be created using **storage.NewClient(ctx)**. The csv file is retrieved as a slice of bytes, and I use bytes.NewReader to implement io.Reader which is the accepted value for csv.NewReader. Then we loop to read all the lines and turn those into objects.

{{< code language="go" isCollapsed="false" >}}
func getFileFromGCS(bucket string, filename string) ([]Team, error) {
	ctx := context.Background()
	client, err := storage.NewClient(ctx)
	if err != nil {
		panic("Unable to create the storage client")
	}
	bkt := client.Bucket(bucket)
	obj := bkt.Object(filename)
	r, err := obj.NewReader(ctx)
	if err != nil {
		panic("cannot read object")
	}
	defer r.Close()

	reader := csv.NewReader(r)

	var teams []Team
	// Loop through lines & turn into object
	for {
		line, error := reader.Read()
		if error == io.EOF {
			break
		} else if error != nil {
			log.Fatal(error)
		}
		teamName := line[0]
		player := line[1]
		nationality := line[2]
		position := line[3]
		appearences, err := strconv.Atoi(line[4])
		if err != nil {
			panic(err)
		}
		goals, err := strconv.Atoi(line[5])
		if err != nil {
			panic(err)
		}
		assists, err := strconv.Atoi(line[6])
		if err != nil {
			panic(err)
		}
		passes, err := strconv.Atoi(line[7])
		if err != nil {
			panic(err)
		}
		interceptions, err := strconv.Atoi(line[8])
		if err != nil {
			panic(err)
		}
		tackles, err := strconv.Atoi(line[9])
		if err != nil {
			panic(err)
		}
		fouls, err := strconv.Atoi(line[10])
		if err != nil {
			panic(err)
		}
		price, err := strconv.Atoi(line[11])
		if err != nil {
			panic(err)
		}

		squand := Team{
			Team:          teamName,
			Player:        player,
			Nationality:   nationality,
			Position:      position,
			Appearences:   appearences,
			Goals:         goals,
			Assists:       assists,
			Passes:        passes,
			Interceptions: interceptions,
			Tackles:       tackles,
			Fouls:         fouls,
			Price:         price,
		}

		teams = append(teams, squand)

	}

	return teams, nil
}
{{< /code >}}

The populated slice of structs can be inserted in our database of choice for persistence. In this case I opted for a cloud based NoSQL database, named Firestore.

### Cloud Firestore:

Cloud Firestore is a flexible, scalable NoSQL database for mobile, web, and server development from Firebase and Google Cloud Platform. Initially, I was looking to Cloud Datastore, but Google recommends instead to use Firestore. Cloud Firestore is the newest version of Cloud Datastore and introduces several improvements over Cloud Datastore. It is out of the scope of this post, to make a thorough comparison between these services, but what may worth mentioned:

* queries become strongly consistent
* transactions are no longer limited to 25 entity groups.
* writes to an entity group are no longer limited to 1 per second

Firestore is a powerful database as it supports flexible/hierarchical data structures, has expressive querying  to retrieve individual, specific documents or to retrieve all the documents in a collection. It leverage GCP infrastructure, it is a managed service and it scales out and in based application demand.

**Cloud Firestore data model:**  
Cloud Firestore is a NoSQL, document-oriented database. Unlike a SQL database, there are no tables or rows. Instead, you store data in documents, which are organized into collections.
Each document contains a set of key-value pairs. All documents must be stored in collections. Documents can contain subcollections and nested objects, both of which can include primitive fields like strings or complex objects like lists.
The documentation can be found [here](https://firebase.google.com/docs/firestore) and also on the [cloud docs](https://cloud.google.com/firestore/docs/), and includes examples for your language of choice.

{{< image src="/img/2019/structure-data.png" style="border-radius: 8px;" >}}

Cloud Firestore is also available in native Node.js, Java, Python, and Go SDKs.
The package https://godoc.org/cloud.google.com/go/firestore provides a client for reading and writing to a Cloud Firestore Database.  
Writing data to a firestore Collection is as simple as you can see below. Instantiate the client, select the collection, in my case “Teams” and range over the slice of structs and create the docs with its fields.

{{< code language="go" isCollapsed="false" >}}
func insertInFirestore(teams []Team) error {
	ctx := context.Background()
	client, err := firestore.NewClient(ctx, "apps-microservices")
	if err != nil {
		log.Printf("cannot create new firestore client: %v", err)
	}
	defer client.Close()

	teamsCol := client.Collection("Teams")
	for _, indTeam := range teams {
		ca := teamsCol.Doc(indTeam.Team + "_" + strings.Replace(indTeam.Player, " ", "_", -1))
		_, err = ca.Set(ctx, Team{
			Team:          indTeam.Team,
			Player:        indTeam.Player,
			Nationality:   indTeam.Nationality,
			Position:      indTeam.Position,
			Appearences:   indTeam.Appearences,
			Goals:         indTeam.Goals,
			Assists:       indTeam.Assists,
			Passes:        indTeam.Passes,
			Interceptions: indTeam.Interceptions,
			Tackles:       indTeam.Tackles,
			Fouls:         indTeam.Fouls,
			Price:         indTeam.Price,
		})

	}

	return nil

}
{{< /code >}}

The complete script is available on [GITHUB](https://github.com/danrusei/microservices_project/tree/master/section_1)

### Deploy the Function:

I’m using modules for dependency management, but as I’m still in the working path so I have to enable the modules.

{{< code language="bash" isCollapsed="false" >}}
export GO111MODULE=on
go mod init
go mod tidy
{{< /code >}}

I’m deploying the function from my local machine using gcloud command-line tool. When using the command-line tool, Google Cloud Functions packages and uploads the contents of your function's directory to a Cloud Storage bucket for you.

{{< code language="bash" isCollapsed="false" >}}
gcloud functions deploy ToFirestore --runtime go111 --trigger-resource premier_league --trigger-event google.storage.object.finalize
{{< /code >}}

* ToFirestore: is the name of the Cloud Function you are deploying
* --runtime go111: is the runtime, in this case Go1.11
* --trigger-resource premier_league: is the trigger resource of this function. The trigger resource specifies the resource for which the trigger event is being observed. In this case, the resource is the name (premier_league) of the Cloud Storage bucket that triggers the function.
* --trigger-event google.storage.object.finalize: is the trigger event for this function, which specifies which action should trigger the function. In this case, the event is google.storage.object.finalize.

The logs are needed to troubleshoot and monitor function calls. Can be retrieved using following gcloud command.

{{< code language="bash" isCollapsed="false" >}}
gcloud functions logs read  --limit=50
{{< /code >}}

### Conclusion

Setting up a data ingestion flow it is not too complicated and the good part is that you don’t have to worry about infrastructure, instead focus on what brings value to your business. The infrastructure is secure and scalable, and the application can benefit from auto-scaling and monitoring by default. Now that I have the data in place I move on, and in the next post I detail the microservices architecture used in my project. 
