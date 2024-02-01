---
title: "DialogFlow Webhook in GO"
date: 2019-10-14
draft: false
tags: ["Dialogflow"]
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

In this article, I make the assumption that you know how to setup the DialogFlow agent by using the intents and entities. I'll just give a short description of the DialogFlow without providing too much details on how to set it up.  
This is the third article and last of the series, where I show another options to consume the data from a REST service. If in the first two posts, I covered the standard web pages and the mobile application using Flutter, this time I'll do the third option which is by Voice.

## What is DialogFlow

DialogFlow is a Google-owned developer of human–computer interaction technologies based on natural language conversations. Its role is to help building  engaging voice and text-based conversational interfaces, such as voice apps and chatbots, powered by Machine Learning. If you are not familiar with DialogFlow and what it provides, there is a very informative playlist on youtube, [Deconstructing Chatbots](https://www.youtube.com/playlist?list=PLIivdWyY5sqK5SM34zbkitWLOV-b3V40B), which cover the basics but also goes into details on how to setup the Intents, Entities for the agent and many more.

Before going into webhook details, this is the architecture diagram. The mobile or the Google Home devices are using Google assistant to talk to DialogFlow. In the DialogFlow the **Webhook call** for the intent is enabled and is pointing to the URL provided by Cloud Function. The [**Cloud Function**](https://cloud.google.com/functions/) talks over the http with the backend application, which is deployed on [**Cloud Run**](https://cloud.google.com/run/).

![DialogFlow diagram](/img/2019/dialogflowDiagram.png)

### GO WebHook script

The below type is identical with the one from the backend, doesn't have to be, but for brevity I'm using the same structure. Checkout the first blog post of the series for an explanation of the fields used within the type.

{{< code language="go" isCollapsed="false" >}}
//Item holds the info retrieve from the API
type Item struct {
	ID        string `json:"id"`
	Created   string `json:"Created"`
	Name      string `json:"name"`
	Expdate   string `json:"expdate"`
	Expopen   int    `json:"expopen"`
	Comment   string `json:"comment"`
	Targetage string `json:"targetage"`
	Isopen    bool   `json:"isopen"`
	Opened    string `json:"opened"`
	Isvalid   bool   `json:"isvalid"`
	Daysvalid int    `json:"daysvalid"`
}
{{< /code >}}

The below code is to retrieve the data from the application backend and select the items of interest. **GetAllItems** is nothing more than a http get request and the retrieved data is unmarshaled into a slice of Items.
**SelectItems** is getting the age type (child or adult) and the item name value and is iterating over the Items to select the ones of interest, adding them into a slice of maps.

{{< code language="go" isCollapsed="false" >}}
//SelectItems select the items from the list
func SelectItems(age string, med string) []map[string]string {
	timeout := time.Duration(5 * time.Second)
	client := http.Client{
		Timeout: timeout,
	}

	url, err := url.Parse("https://yours_backend_server/")
	if err != nil {
		log.Fatalln("could not parse the URL")
	}

	items, err := GetAllItems(&client, url)
	if err != nil {
		log.Fatalf("something happened while trying to retrive data: %v", err)
	}

	listItems := []map[string]string{}

	for _, item := range items {
		if strings.Contains(strings.ToLower(item.Name), strings.ToLower(med)) {
			findItem := make(map[string]string)
			if item.Targetage == age {
				findItem["name"] = item.Name
				findItem["expiredays"] = strconv.Itoa(item.Daysvalid)

				listItems = append(listItems, findItem)
			}
			continue
		}
		continue
	}

	return listItems
}

//GetAllItems retrieve the items from the service
func GetAllItems(client *http.Client, url *url.URL) ([]Item, error) {

	req, err := http.NewRequest("GET", url.String(), nil)
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")

	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}

	defer resp.Body.Close()

	var items = []Item{}

	err = json.NewDecoder(resp.Body).Decode(&items)
	if err != nil {
		return nil, err
	}

	return items, err
}
{{< /code >}}

Until now, nothing was specific to Dialogflow. What we have implemented so far is the communication between the Google Cloud Function, where the webhook script is hosted, and the backend app which runs on Cloud Run. Moving forward we need to understand how the DialogFlow communicates with Cloud Function.  
As I have alluded before, the webhook call should be enabled for the Dialogflow **Intent** and point towards the Cloud Function URL. Dialogflow send a [**WebhookRequest**](https://godoc.org/google.golang.org/genproto/googleapis/cloud/dialogflow/v2#WebhookRequest) and expect to receive back a [**WebhookResponse**](https://godoc.org/google.golang.org/genproto/googleapis/cloud/dialogflow/v2#WebhookResponse). In order to facilitate the communication we are using the [GO code](https://godoc.org/google.golang.org/genproto/googleapis/cloud/dialogflow/v2) which was generated from the protocol buffer definition. The problem is that the generated Go code corresponds to the protocol buffer, but we'll get/send JSON requests.
Package [**jsonpb**](https://godoc.org/github.com/golang/protobuf/jsonpb) provides marshaling and unmarshalling between protocol buffers and JSON. This package produces a different output than the standard "encoding/json" package, which does not operate correctly on protocol buffers.

{{< code language="go" isCollapsed="false" >}}
import (
	"github.com/golang/protobuf/jsonpb"
	"google.golang.org/genproto/googleapis/cloud/dialogflow/v2"
)
{{< /code >}}

Now as we can unmarshall the requests into **dialogflow.WebhookRequest**, we are able to get all the data of interest like **Intent name** and the **Parameters**. Basically the webhook can be enabled on any "Intent" and the traffic will be sent to the same URL. Here I'm using two Intents: "itemstock" and "listitems".  
The parameters are the **Entities**, which is kindly a named group for a list of items. Based on the official definition:
"Entities are powerful tools used for extracting parameter values from natural language inputs. Any important data you want to get from a user’s request, will have a corresponding entity."  
Performing **req.GetQueryResult().GetParameters().GetFields()** request you'll get a map[string]*Value. **"Value"** represents a dynamically typed value which can be either null, a number, a string, a boolean, a recursive struct value, or a list of values. A producer of value is expected to set one of that variants, absence of any variant indicates an error. As you can see below we call **GetStringValue()** whish is a simple method:

{{< code language="go" isCollapsed="false" >}}
func (m *Value) GetStringValue() string {
	if x, ok := m.GetKind().(*Value_StringValue); ok {
		return x.StringValue
	}
	return ""
}
{{< /code >}}

Finally, after we call "SelectItems" function and receive the wanted data, we have to construct the response. **fullfilement** is a slice of strings, which increase in size based on the number of items. We construct the response within a **dialogflow.WebhookResponse** struct and send over the wire to DialogFlow.

{{< code language="go" isCollapsed="false" >}}
//F is the entry point for cloud functions
func F(w http.ResponseWriter, r *http.Request) {

	defer r.Body.Close()
	req := dialogflow.WebhookRequest{}
	if err := jsonpb.Unmarshal(r.Body, &req); err != nil {
		log.Println("Couldn't Unmarshal request to jsonpb")
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	log.Println("A number of longs below to test")
	log.Println("Get parameters: ", req.GetQueryResult().GetParameters().GetFields())
	log.Println("Get Actions: ", req.GetQueryResult().GetAction())
	log.Println("Get Intent: ", req.GetQueryResult().GetIntent())

	mapfields := req.GetQueryResult().GetParameters().GetFields()

	items := []map[string]string{}

	switch req.QueryResult.Intent.DisplayName {
	case "itemstock":
		log.Println("It was requested something about itemstock")
		age := mapfields["AgeType"].GetStringValue()
		med := mapfields["meds"].GetStringValue()
		items = SelectItems(age, med)

	case "listitems":
		log.Println("It was requested something about listitems")
		age := mapfields["AgeType"].GetStringValue()
		medAll := ""
		items = SelectItems(age, medAll)
	}

	fullfilement := []string{}

	for _, item := range items {
		text := fmt.Sprintf("%s that expires in %s days.", item["name"], item["expiredays"])
		fullfilement = append(fullfilement, text)
	}

	response := dialogflow.WebhookResponse{}

	if len(fullfilement) > 0 {
		response = dialogflow.WebhookResponse{
			FulfillmentText: fmt.Sprintf("You have these meds back home: " + strings.Join(fullfilement, "; \n")),
		}
	} else {
		response = dialogflow.WebhookResponse{
			FulfillmentText: fmt.Sprintf("Sorry, you don't have these meds back home"),
		}
	}

	data, err := json.MarshalIndent(response, "", " ")
	if err != nil {
		log.Printf("Can't marshall the data %v: ", err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(data)

}
{{< /code >}}

### Deploy the script

As i mentioned before, this script is deployed on Google Cloud Function.

{{< code language="bash" isCollapsed="false" >}}
gcloud functions deploy [function_name] --entry-point F --runtime go111 --region [your_region] --trigger-http
{{< /code >}}

## Conclusion

This is a simple example, Dialogflow is a reach environment which lets you construct very complex chatbots, but if you are looking for a starting point, this post can be useful. The [complete code](https://github.com/danrusei/dev-state_blog_code/tree/master/assistant_webhook) is available on Github.
