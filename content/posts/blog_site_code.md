---
title: "Part 2: Blog Site written in GO"
date: 2019-05-20
draft: false
tags: ["Blog", "App Engine"]
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

This is the second post from a series of three where I present an option to build a blog site.
This time I created a simple blog site in GO, mostly using standard library and a couple of external packages. Althouth it is simple it can serve specific needs. If you don't need a fully featured site but fast and reliable this may work for you. It was designed with the minimal amount of features required for the blog site and can be developed further on.

### Build

The requirements:

* generate the blog post html files from markdown content  
* extract metadata from the json file that accompanies the blog post markdown file
* single-page site, that lists the blog posts and the info gathered from metadata  

The code is published on [GitHub](https://github.com/danrusei/dev-state_blog_code/tree/master/blog_site_code) and you can freely use it in your projects. If you want to try it out, it is temporary hosted on this link : https://apps-devel.appspot.com/, it may change in future.  
Beside the standard library several packages were instrumental in building the site:  
**[Gorilla Mux](https://github.com/gorilla/mux)**, it is a powerfull yet fast request router and dispatcher for matching incoming requests to their respective handlers.  
**[BlackFriday](https://github.com/russross/blackfriday)** is a well known Markdown rendering engine.  
**[Chroma](https://github.com/alecthomas/chroma)**, is a fast syntax highlighting which support a large number of languages.  
Both, Blackfriday and Chroma are used by default by Hugo, the framework I used in [Part 1](https://dev-state.com/blog/hugo_blog_static/) of the post series.

I'm not in favour of reinveting the wheel, so before starting this project I did some reasearch on the web. I found a couple of resources from where I borrowed several concepts, like in browser cache control and parsing the html to highlight the code.  I got inspired in my project by [blog generator](https://github.com/zupzup/blog-generator) , which is a very lightweight static blog generator and the [blog site code](https://github.com/ryanrolds/pedantic_orderliness) posted by Ryan Olds.

**The Project Structure** is composed by few folders. The entry point is the **main.go** that holds the handlers and instantiate the script components. **Content** directory holds the markdown files with associated json metadata. **Templates** holds the html templates for posts and main page. Blog post images are placed into **Images** folder. The site engine is within **Site** folder.

The operational model is simple, all the rendered markdown blog posts/pages along with static files are stored in a map structure. Once the program is compiled and started all the files will be served from memmory. 

{{< code language="go" isCollapsed="false" >}}
// ContentCache create a key, page relation
type ContentCache map[string]interface{}
{{< /code >}}

When the request is comming, the content is searched in store by the key and if available the page will be presented to user. I'm using [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) and [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) header fields which instruct the browser about caching mechanisms. Caching is a technique that stores a copy of a given resource and serves it back when requested. When a web cache has a requested resource in its store, it intercepts the request and returns its copy instead of re-downloading from the originating server. It improve the site performance and optimize the site network traffic. 

{{< code language="go" isCollapsed="false" >}}
router := mux.NewRouter()
router.HandleFunc("/{key}", s.postHandler).Methods("GET")

//handlers as a method to siteconfig struct
func (s *SiteConfig) postHandler(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	key := vars["key"]

	// Try to get cache entry for post
	post := s.posts.Get(key)
	if post == nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}

	if r.Header.Get("If-None-Match") == post.Etag {
		w.WriteHeader(http.StatusNotModified)
		return
	}

	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Header().Set("Cache-Control", "public, must-revalidate")
	w.Header().Set("Etag", post.Etag)
	w.WriteHeader(http.StatusOK)
	w.Write(post.Content)
}
{{< /code >}}

The simplified flow is as follows. It iterate over the blog post folders, unmarshal json metadata and generate html file for each markdown file using blackfriday. Then I'm using [this script](https://github.com/zupzup/markdown-code-highlight-chroma) for highlighting the code inside the blog post using the chroma library. Finally I make use of html/template standard package to generate HTML output, safe against code injection.  

{{< code language="go" isCollapsed="false" >}}
for _, path := range p.Paths {
		fmt.Printf("\tGenerating Post : %s...\n", path)
		post, err := newPost(path, templ)
		if err != nil {
			return err
		}
		p.Cache.Set(post.Name, post)
		fmt.Printf("\tFinished generating Post: %s...\n", post.Meta.Title)
    }
{{< /code >}}

If you want to try it out on your local machine, clone [the repository](https://github.com/danrusei/dev-state_blog_code/tree/master/blog_site_code), modify the content and build the binary.

{{< code language="bash" isCollapsed="false" >}}
$ cd blog_site_code/
$ go build
{{< /code >}}

The **go build** command lets you build an executable file for any Go-supported target platform. Cross-compiling works by setting required environment variables that specify the target operating system and architecture. **GOOS** for target operating system and **GOARCH** for targeted architecture. 

{{< code language="bash" isCollapsed="false" >}}
// For Windows:
$ env GOOS=windows go build -o binary_name
// For Mac:
$ env GOOS=darwin go build -o binary_name
{{< /code >}}

### Deploy

The project will be deployed on App Engine. [App Engine](https://cloud.google.com/appengine/) is part of Google Cloud Platform and it is fully managed serverless application platform. The goal is the user to focus just on writing code, without the worry of managing the underlying infrastructure. With capabilities such as automatic scaling-up and scaling-down of your application. It support a number of programming languages like: Java, PHP, Node.js, Python, C#, .Net, Ruby and Go or you can bring your own language runtime.

Ensure you have the [Google Cloud SDK](https://cloud.google.com/sdk/install) installed on your system.
Additionaly **app-engine-go** component has to be installed as well.

{{< code language="bash" isCollapsed="false" >}}
$ gcloud components install app-engine-go

// check it out:
$ gcloud components list
{{< /code >}}

Then run **gcloud config list** to ensure that you are logged in with right account and project.
Initialize your App Engine app with the project and choose its region.

{{< code language="bash" isCollapsed="false" >}}
$ gcloud app create --project=[YOUR_PROJECT_NAME]
{{< /code >}}

Create app.yaml file with following content.

{{< code language="yaml" isCollapsed="false" >}}
runtime: go111

handlers:
# Configure App Engine to serve any static assets.
- url: /images
  static_dir: images

# Use HTTPS for all requests.
- url: /.*
  secure: always
  redirect_http_response_code: 301
{{< /code >}}

Last step is to deploy the application on App Engine.

{{< code language="bash" isCollapsed="false" >}}
$ gcloud app deploy
{{< /code >}}

You'll get back a link like https://[YOUR_PROJECT_ID].appspot.com if the deployment succed.


### Conclusion

It was fun to write this blog site from scratch, it was a good learning experience. It can get easily very complex if you integrate more features. The advatage is that it is tailored for your specific need, and does not include unnecessary packages. The code is light and fast but it takes time and many iterations to develop it.