---
title: "Part 3: Blog Site with Buffalo"
date: 2019-05-27
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

This is the third and last post from a series of three where I present an option to build a blog site. In the first 2 posts I showed you how to create a simple [static site with Hugo](http://localhost:1313/blog/hugo_blog_static/) and how to build your [own blog site](http://localhost:1313/blog/blog_site_code/) using mainly standard library. In this post I'll build the blog site using one of the best GO web frameworks out there, [Buffalo](https://gobuffalo.io/en).

### Build

**What is Buffalo?**  
Buffalo is a Go web development eco-system, designed to make the life of a Go web developer easier. Buffalo helps you to generate a web project that already has everything from front-end (JavaScript, SCSS, etc.) to back-end (database, routing, etc.) already hooked up and ready to run. Buffalo generate a directory structure and it is based on MVC pattern, where **action** folder is the Controller part, **models** folder is the Model part and the **templates** folder is the View part of the MVC.

{{< image src="/img/2019/buffalo.jpg" style="border-radius: 8px;" >}}

**Install Buffalo**

The install guides, for different platforms, are very well documented on [Buffalo site](https://gobuffalo.io/en/docs/getting-started/installation/), I'm running linux. By default Buffalo requires a database setup, and it has a deep integration with [Pop](https://godoc.org/github.com/gobuffalo/pop), which wraps [sqlx](https://github.com/jmoiron/sqlx) library and gives another level of abstraction. It makes easy to do CRUD operations, run migrations, and build/execute queries. Pop supports PostgreSQL which is the default option, CockroachDB,MySQL, SQLite3.

I'm using PosgreSQL, but I would rather run it from a docker container instead of installing it on my system. However, I want the database information to persist, therefore I'll mount /var/lib/postgresql/data to a local directory.

{{< code language="bash" isCollapsed="false" >}}
docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v $HOME/docker_volumes/postgres:/var/lib/postgresql/data  postgres
{{< /code >}}

To install gobuffalo you need a working Go environment with configured $PATH. Also nodejs and npm or yarn are required to build the asset pipeline, built upon [webpack](https://github.com/webpack/webpack).
I'm not too familiar with Frontend development, you can find a plenty of resources on the Buffalo site.

{{< code language="go" isCollapsed="false" >}}
go get -u -v github.com/gobuffalo/buffalo/buffalo
{{< /code >}}

Above command, install buffalo and its dependencies. If everything was installed properly you should you should be able to see below message while running buffalo.

{{< code language="bash" isCollapsed="false" >}}
$ buffalo
//Helps you build your Buffalo applications that much easier!
//.....
//Should also contain Usage and Available commands
{{< /code >}}

**Create the site**

Next, I create the App, with a simple command:

{{< code language="bash" isCollapsed="false" >}}
$ buffalo new appname
{{< /code >}}

It will generates all the files and download the dependencies. You need to open up the "database.yml" file and edit it with the correct entitlements, as appropriate for your environment. Create the databases using below command:

{{< code language="bash" isCollapsed="false" >}}
$ buffalo pop create -a
{{< /code >}}

Buffalo ships with a command that will watch your application and automatically rebuild the Go binary and any assets for you:

{{< code language="bash" isCollapsed="false" >}}
$ buffalo dev
{{< /code >}}

Open the link [http://127.0.0.1:3000](http://127.0.0.1:3000) in your browser and you should be able to see the "Welcome to Buffalo!" page.
Now you can start and develop the application. I prefer to use plugins if available to generate the necessary code. One of them is **buffalo-goth** which generates a full auth implementation. This adds a bunch of boilerplate code in auth.go file, including a couple of Middlewares **SetCurrentUser** that set current_user in buffalo context and **Authorize** that allows users to access a specific page. [Gorilla sessions](https://github.com/gorilla/sessions) package is used to verify user authetication. First you need to download and install the package:

{{< code language="bash" isCollapsed="false" >}}
$ go get -u github.com/gobuffalo/buffalo-goth
$ buffalo plugins install github.com/gobuffalo/buffalo-goth

Check If it is succefull installed:
$ buffalo plugins list
{{< /code >}}

Package goth provides a simple, clean, and idiomatic way to write authentication packages for Go web applications. It support a large number of providers like facebook, twitter, linkedin and many more. Following command generates users and routes and also fizz migrations.

{{< code language="bash" isCollapsed="false" >}}
$ buffalo genarate goth-auth google facebook
{{< /code >}}

Fizz is a common DSL for migrating databases and it tries to be as database-agnostic as possible. This is the language used by default by Pop to define database migrations.

{{< code language="bash" isCollapsed="false" >}}
./migrations/20190527092032_create_users.down.fizz
./migrations/20190527092032_create_users.up.fizz
{{< /code >}}

Once migration files have been created, you can run:

{{< code language="bash" isCollapsed="false" >}}
$ buffalo pop migrate
{{< /code >}}

Next, I followed a simplified version of [BLog App tutorial](https://gobuffalo.io/en/docs/examples#blog-app), as this is just a sample toy project and not a full featured site.
I need an index page to list all blog posts summaries and to be able to view the blog post. I skiped post edit and creation features as the content is imported from markdown files.

To be productive and minimize the need to write boilerplate code, Buffalo provides you actions and resources generator. Resource generator is the most complete as it creates  models, actions and templates for you. Instead action generator it creates only actions, templates and register the actions with the app but does not include the model part, that can be manually created later on.  

{{< code language="bash" isCollapsed="false" >}}
$ buffalo generate actions posts index detail
{{< /code >}}

Files automatically generated:

{{< code language="bash" isCollapsed="false" >}}
./actions/posts.go
./actions/posts_test.go
./templates/detail.html
./templates/index.html
{{< /code >}}

and registered within app.go file with the specific handlers.

{{< code language="go" isCollapsed="false" >}}
app.GET("/posts/index", PostsIndex)
app.GET("/posts/detail/{pid}", PostsDetail)
{{< /code >}}

Create the Post struct, within the models directory:

{{< code language="go" isCollapsed="false" >}}
type Post struct {
	ID        uuid.UUID `json:"id" db:"id"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
	Title     string    `json:"title" db:"title"`
	Content   string    `json:"content" db:"content"`
}

type Posts []Post
{{< /code >}}

Next you have to generate the migrations files, fill in with proper information and apply the migration. 

{{< code language="bash" isCollapsed="false" >}}
$ buffalo pop generate fizz create_posts

create  migrations/20190527123730_create_posts.up.fizz
create  migrations/20190527123730_create_posts.down.fizz

//fill in the files with drop_table and create_table

$ buffalo pop migrate up
{{< /code >}}

To view the posts, we need PostsIndex and PostsDetail handlers completed

{{< code language="go" isCollapsed="false" >}}
func PostsIndex(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	posts := &models.Posts{}
	// Paginate results. Params "page" and "per_page" control pagination.
	// Default values are "page=1" and "per_page=20".
	q := tx.PaginateFromParams(c.Params())
	// Retrieve all Posts from the DB
	if err := q.All(posts); err != nil {
		return errors.WithStack(err)
	}
	// Make posts available inside the html template
	c.Set("posts", posts)
	// Add the paginator to the context so it can be used in the template.
	c.Set("pagination", q.Paginator)
	return c.Render(200, r.HTML("posts/index.html"))
}

func PostsDetail(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	post := &models.Post{}
	if err := tx.Find(post, c.Param("pid")); err != nil {
		return c.Error(404, err)
	}
	c.Set("post", post)
	comment := &models.Comment{}
	c.Set("comment", comment)
	comments := models.Comments{}
	if err := tx.BelongsTo(post).All(&comments); err != nil {
		return errors.WithStack(err)
	}
	for i := 0; i < len(comments); i++ {
		u := models.User{}
		if err := tx.Find(&u, comments[i].AuthorID); err != nil {
			return c.Error(404, err)
		}
		comments[i].Author = u
	}
	c.Set("comments", comments)
	return c.Render(200, r.HTML("posts/detail"))
}
{{< /code >}}

Logged in users are allowed to comment on the posts, therefore I included "Creating Comments" part of the [tutorial](https://mmikael.com/posts/buffalo-part2/).

Last step is to load the content, create the **Content** folder and copy the markdown files within it. Because the site is using PostgreSql as a backend database I had to populate it with the content from the markdown *.md files. To do this you have to create a task . Tasks are small scripts that are often needed when writing an application. These tasks might be along the lines of seeding a database, parsing a log file, or even a release script. Buffalo uses the grift package to make writing these tasks simple.
You can find more information [here](https://gobuffalo.io/en/docs/tasks).

{{< code language="go" isCollapsed="false" >}}
var _ = grift.Namespace("db", func() {

	grift.Desc("seed", "Seeds a database")
	grift.Add("seed", func(c *grift.Context) error {
		path := "content/"

		files, err := ioutil.ReadDir(path)
		if err != nil {
			return err
		}

		for _, file := range files {
			name := file.Name()
			content, err := ioutil.ReadFile(path + name)
			if err != nil {
				return err
			}
			u := &models.Post{
				Title:   name,
				Content: string(content),
			}
			err = models.DB.Create(u)
		}
		return err
	})
})
{{< /code >}}

The blog script is not yet finalized as I have not tested all its functionalities, also it may require a site admin privileged user to manage all the comments. Right now only the owners are able to edit/delete their own comments. Also we may want to improve the readibility of the post code, by including the bfchroma syntax highlighter of the Blackfriday renderer as a template handler.
The complete code is available on [Github](https://github.com/danrusei/dev-state_blog_code/tree/master/blog_site_buf) and you can play with it and use it for your needs.

### Deploy

For deployment, I opted for a lift and shift solution. Compute Engine offers the flexibility I needed but it comes with a cost, it's in your responsibility to configure, manage, and monitor the system. Google ensures that resources are available, reliable, and ready for you to use, but it's up to you to provision and manage them. You can use VMs, named instances, to deploy your app,much like you would if you had your own hardware infrastructure. In this case I used an Ubuntu image runnig on a single vCPU machie (n1-standard-1).
Providing the framework and best practices to set up a website is beyond the scope of this tutorial, but if this would be a business critical site and you would need a highly available infrastructure then you should carefully plan the design:

* consider adopting managed storage services for storing the data. [Cloud SQL](https://cloud.google.com/sql/docs/) may be a good option as it offers fully managed MySQL and ProstgreSQL databases.
* use load-balancing technologies to distribute the workload among servers. There are several options, well known are [HTTP(S) load balancing](https://cloud.google.com/load-balancing/docs/https/) and [Network load balancing](https://cloud.google.com/load-balancing/docs/network/)
* [Autoscaling with Compute Engine](https://cloud.google.com/compute/docs/autoscaler/), you can set up your architecture to automatically add and remove servers as traffic pattern change.            

### Conclusion

This does not even scratch the surface of how many things Buffalo can offer, it's an amazing framework which I would strongly recommend for those who are creating a complex, enterprise level and interactive Web Apps with GO. Maybe it was a little bit too much for this small project, but still something worth to consider. If you are creating a static site or a small presentation page you could stick with [Hugo](https://dev-state.com/blog/hugo_blog_static/). Also, if you enjoy coding in GO and you want to have some fun, checkout my [second post](https://dev-state.com/blog/blog_site_code/) of this series.
