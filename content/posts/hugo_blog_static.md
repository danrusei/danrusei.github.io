---
title: "Part 1: Blog Site with Hugo"
date: 2019-05-10
draft: false
categories: ["Go"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

This is the first post from a series of three where I present an option to build a blog site.
Hopefully, this series will give you some hints on what it takes to build a simple site/blog.  
* Although my goal is to code the blog site in GO, I want to give a try to this wonderfull tool, Hugo. If you don't have time to code and need a full featured static site this is one of the best solution out there in the market. However in the next posts will dive more into code.

### Build

**What is Hugo?**   
Hugo is one of the most popular open-source static site generators. With its amazing speed and flexibility, Hugo makes building websites fun again. Hugo is written in Go with support for multiple platforms, but you don’t need to install Go to enjoy Hugo.

{{< image src="/img/2019/hugo.jpg" style="border-radius: 8px;" >}}

The best part is that you don't have to know any programming languange to build your own blog site.
All files are generated by Hugo and copied into **public** folder. Once done, the site is ready to be hosted on your local or cloud infrastructure. 

**Install Hugo**

Hugo can run and be installed on multiple platforms. The install process is very well covered at: https://gohugo.io/getting-started/installing. I'm using Linux so I opted for a snap package. Extended option is needed for Sass/SCSS support but can be opted out by removing the flag.

{{< code language="bash" isCollapsed="false" >}}
$ snap install hugo --channel=extended
{{< /code >}}

**Create the site**

{{< code language="bash" isCollapsed="false" >}}
$ hugo new site site_name
{{< /code >}}

Add a theme. Plenty to choose from, but I'm sticking with [Future Imperfect Slim](https://themes.gohugo.io/hugo-future-imperfect-slim/),  which is a modern and clean theme.

{{< code language="bash" isCollapsed="false" >}}
$ cd themes/
$ git clone https://github.com/pacollins/hugo-future-imperfect-slim.git
$ cd ..
{{< /code >}}

Most themes comes with an **exampleSite**, where you can check out the structure and more important has a **config.toml** configuration file which can be copied and tweaked for your needs.

{{< code language="bash" isCollapsed="false" >}}
$ cp themes/hugo-future-imperfect-slim/exampleSite/config.toml .
{{< /code >}}

To create the content is simple as well. The markdown files are created within **content** folder and can be edited with your favorite text editor. I would recommed something that understand markdown files.
If you are not familiar with editing markdown, this [Markdown Cheatsheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) can help you get off the ground.

{{< code language="bash" isCollapsed="false" >}}
$ hugo new about/_index.md
$ hugo new blog/examples.md
{{< /code >}}

The changes can be seen in real time, Hugo has a builtin http server. It will build the site and make it available at localhost:1313.

{{< code language="bash" isCollapsed="false" >}}
$ hugo server
{{< /code >}}

Lastly, **hugo** command builds the site by generating the html files and copy all the static content, including images, css & js in the **public** folder.

{{< code language="bash" isCollapsed="false" >}}
$ hugo
{{< /code >}}

### Deploy

**Host the site on Google Cloud Storage** 

Finding a hosting service it is trivial these days, I believe most of the Big Cloud providers offer several options to host a site. Definitely, I searched for free hosting alternatives to host my small project. I opted for **Cloud Storage** from Google Cloud Platform. As part of the GCP Free Tier, Cloud Storage provides resources that are free to use up to specific limits.  
A valid GCP account is needed and the Web Console or Cloud SDK can be used to setup the environment. The configuration and instalation of Cloud SDK is out of the scope of this tutorial,
however here are some thoroughly [Quickstarts](https://cloud.google.com/sdk/docs/quickstarts) to start with.

**Caution:** Only non-secure HTTP sites can be hosted on Cloud Storage. However to serve custom domains over SSL, you can use Cloud Storage bucket as a [load balancer backend](https://cloud.google.com/load-balancing/docs/https/adding-a-backend-bucket-to-content-based-load-balancing), use a third party [Content Delivery Network](https://cloudplatform.googleblog.com/2015/09/push-google-cloud-origin-content-out-to-users.html) or serve the static website content from [Firebase Hosting](https://firebase.google.com/docs/hosting) instead of Cloud Storage which I plan to explain in last section of this tutorial.  

**Prerequisites:**  

* [Verify](https://cloud.google.com/storage/docs/domain-name-verification#verification) the ownership of the domain in case you want to create a bucket that uses the domain name.  
* Create the CNAME record. A CNAME record is a type of DNS record. It directs traffic that requests a URL from your domain to the resources you want to serve, in this case objects in your Cloud Storage buckets. I'm using GoDaddy and I have added a Type: CNAME , Name: www, Value: c.storage.googleapis.com as a new dns record.

Below are the steps to set it up but I would recommend to read the [Official Guide](https://cloud.google.com/storage/docs/hosting-static-website), which detail the steps.

Create the storage bucket where the site will be hosted.

{{< code language="bash" isCollapsed="false" >}}
$ gsutil mb gs://www.your_site_name.com
{{< /code >}}

* replace www.your_site_name.com with the real site name

Upload site's files, in this case, hugo generated public folder.

{{< code language="bash" isCollapsed="false" >}}
$ gsutil -m rsync -R public gs://www.your_site_name.com
{{< /code >}}

As it is a public site and to allow everyone's acces to site content the objects within the bucket have to be publicly readable.

{{< code language="bash" isCollapsed="false" >}}
$ gsutil iam ch allUsers:objectViewer gs://www.your_site_name.com
{{< /code >}}

Define the site entry point and error page. Assign an index page suffix, which is controlled by the MainPageSuffix property and a custom error page, which is controlled by the NotFoundPage property. Assigning either is optional, but without an index page, nothing is served when users access your top-level site, for example, http://www.your_site_name.com.

{{< code language="bash" isCollapsed="false" >}}
$ gsutil web set -m index.html -e 404.html gs://www.your_site_name.com
{{< /code >}}

**Host the site on Firebase Hosting** 

[Firebase](https://firebase.google.com/) is a comprehensive mobile and web development platform, it was acquired by Google and it is tighly integrate with Google Cloud PLatform. This platform provide many tools to developers build the app fast, without managing infrastructure. If you are familiar with IaaS, PaaS terminologies, Firbase is Backed-as-a-Service (BaaS) which provide an API and tools for different computer languages to integrate with the application backend.  
I switched from  GCP Storage to Firbase Hosting because it offers **Free SSL Certificates** for custom domains.

Log in to the firebase console and create a new project for hosting the static site created with Hugo.
Follow [this procedure](https://firebase.google.com/docs/hosting/custom-domain) to connect your project with a custom domain. It is similar with the one above where you are asked to validate the domain and create the DNS records, except that this time the A records will be pointed to Firebase Hosting.

Node.js has to be installed on the machine. Once installed use node package manager to install **firebase-tools**.

{{< code language="bash" isCollapsed="false" >}}
$ npm install -g firebase-tools
{{< /code >}}

Login to firebase from cli and initialize the project. You have to be in your main hugo project folder.
In the init process, select the **Hosting** option then select your newly created **firebase project**. It asks to select directory that contain the assests to be uploaded, the default option **public** is fine with us. Select **Yes** to configure as a single page app and **No** to overwrite the index.html file.

{{< code language="bash" isCollapsed="false" >}}
$ firebase login
$ firebase init
{{< /code >}}

The site can be tested locally by running firebase serve command

{{< code language="bash" isCollapsed="false" >}}
$ firebase serve
{{< /code >}}

and easily deployed to cloud using deploy command.

{{< code language="bash" isCollapsed="false" >}}
$ firebase deploy
{{< /code >}}

### Conclusion

It takes only a couple of hours to setup everything, starting from site creation to cloud deployment. It allows you to quickly be up and running and focus on blog content creation. Although you have to be comfortable with some cloud concepts and technical enough to be able to install the applications, you don't have to know any programming languages. This is fine, but if you need a custom blog site check out [Part 2](https://dev-state.com/blog/blog_site_code/) and [Part 3](https://dev-state.com/blog/blog_site_buffalo/) of this series. 
