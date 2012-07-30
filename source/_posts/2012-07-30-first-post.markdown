---
layout: post
title: "Quick doens't have to mean dirty"
date: 2012-07-30 11:43
comments: true
categories: 
---

After reading the wonderful article whose title I stole about making
a guestbook app in Flask, I decided to see how it would compare to my
favorite language of the year, Go. So here's my take.

## First Steps

Let's create a new directory to hold the project. I'm gonna host the code on
github so let's make the local directory match the import path.

	sigma:~ zeebo$ cd ~/Code/go/src
	sigma:src zeebo$ mkdir -p github.com/zeebo/gostbook
	sigma:src zeebo$ cd github.com/zeebo/gostbook/
	sigma:gostbook zeebo$ git init
	Initialized empty Git repository in /Users/zeebo/Code/go/src/github.com/zeebo/gostbook/.git/

Note that `~/Code/go` is a directory in my GOPATH environment variable, the only
piece of configuration I need to do to have the build tool know how to fetch and
build any code that uses these conventions. Lets put in a little hello world code.

{% codeblock Hello World - main.go %}
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintln(w, "Hello World!")
}

func main() {
	http.HandleFunc("/", hello)
	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
{% endcodeblock %}

This registers a handler that will match any path and write
`Hello World!` in the response. Building and running this code
runs a server that listens on port 8080, so lets visit it.

	sigma:gostbook zeebo$ go build
	sigma:gostbook zeebo$ ./gostbook &
	[1] 39629
	sigma:gostbook zeebo$ curl localhost:8080
	Hello World!
	sigma:gostbook zeebo$ kill 39629

Neat!

## Commit

Let's do our source control duty, and make a commit with our super simple
app.

	sigma:gostbook zeebo$ cat .gitignore 
	*
	!.gitignore
	!*.go
	!*.html
	sigma:gostbook zeebo$ git status
	# On branch master
	#
	# Initial commit
	#
	# Untracked files:
	#   (use "git add <file>..." to include in what will be committed)
	#
	#	.gitignore
	#	main.go
	nothing added to commit but untracked files present (use "git add" to track)
	sigma:gostbook zeebo$ git add .
	sigma:gostbook zeebo$ git commit -m 'initial commit'
	[master (root-commit) de0b184] initial commit
	 2 files changed, 21 insertions(+)
	 create mode 100644 .gitignore
	 create mode 100644 main.go

## Templates

The next step is to put templates in. Lets make a template directory
and some basic templates in there. I'll steal the templates from
Eevee's post and change them to use the built in `html/template` package
from the standard library. Here's the source:

{% codeblock templates/_base.html %}
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>{{"{{"}} template "title" . {{"}}}}</title>
    </head>
    <body>
        <section id="contents">
            {{"{{"}} template "content" . {{"}}}}
        </section>
        <footer id="footer">
            My Cool Guestbook 2000 © me forever
        </footer>
    </body>
</html>
{% endcodeblock %}

{% codeblock templates/index.html %}
{{"{{"}} define "title" {{"}}}}Guestbook{{"{{"}} end {{"}}}}

{{"{{"}} define "content" {{"}}}}
    <h1>Guestbook</h1>

    <p>Hello, and welcome to my guestbook, because it's 1997!</p>

    <ul class="guests">
        <li>...</li>
    </ul>
{{"{{"}} end {{"}}}}
{% endcodeblock %}

Updating the Go code is a little more work, but not much.

{% codeblock Template World - main.go %}
import (
	"html/template"
	"net/http"
)

var index = template.Must(template.ParseFiles(
	"templates/_base.html",
	"templates/index.html",
))

func hello(w http.ResponseWriter, req *http.Request) {
	if err := index.Execute(w, nil); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
{% endcodeblock %}

Building and running again, we see it's working:

	sigma:gostbook zeebo$ go build
	sigma:gostbook zeebo$ ./gostbook &
	[1] 39918
	sigma:gostbook zeebo$ curl localhost:8080
	<!DOCTYPE html>
	<html lang="en">
	    <head>
	        <title>Guestbook</title>
	    </head>
	    <body>
	        <section id="content">
	            
	    <h1>Guestbook</h1>

	    <p>Hello, and welcome to my guestbook, because it's 1997!</p>

	    <ul class="guests">
	        <li>...</li>
	    </ul>

	        </section>
	        <footer id="footer">
	            My Cool Guestbook 2000 © me forever
	        </footer>
	    </body>
	</html>
	sigma:gostbook zeebo$ kill 39918

Let's be diligent and make another commit. On to data!

## Databases

Go has many database bindings but the one I find easiest to work with
would be MongoDB with the excellent mgo driver. Let's create our data
model.

{% codeblock Database entry - entry.go %}
package main

import (
	"labix.org/v2/mgo/bson"
	"time"
)

type Entry struct {
	ID        bson.ObjectId `bson:"_id,omitempty"`
	Timestamp time.Time
	Name      string
	Message   string
}

func NewEntry() *Entry {
	return &Entry{
		Timestamp: time.Now(),
	}
}
{% endcodeblock %}

We just create a struct with some fields, and for the ID field add some
tags to it to instruct bson to omit it if the value is empty, and name it
`_id` when serializing. We also provide a `NewEntry` function for creating
an Entry at the current time.

Now lets add support to the handler.

{% codeblock Databased up - main.go %}
func hello(w http.ResponseWriter, req *http.Request) {
	//grab a clone of the session and close it when the
	//function returns
	s := session.Clone()
	defer s.Close()

	//set up the collection and query
	coll := s.DB("gostbook").C("entries")
	query := coll.Find(nil).Sort("-timestamp")

	//execute the query
	//TODO: add pagination :)
	var entries []Entry
	if err := query.All(&entries); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	//execute the template
	if err := index.Execute(w, entries); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
}

var session *mgo.Session

func main() {
	var err error
	session, err = mgo.Dial("localhost")
	if err != nil {
		panic(err)
	}

	http.HandleFunc("/", hello)
	if err = http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
{% endcodeblock %}

Interacting with the databse requires a little boilerplate in the handler,
but this can easily be removed by clever use of Go's interfaces. The `net/http`
package will serve anything with a `ServeHTTP(ResponseWriter, *Request)` method,
so you can decorate handlers by wrapping them in simple types that implement
that interface. Doing that is left as an exercise to the reader :)

Here's how we change the template:

{% codeblock templates/index.html %}
    <ul class="guests">
        {{"{{"}} range . {{"}}}}
        <li>
            <blockquote>{{"{{"}} .Message {{"}}}}</blockquote>
            <p>- <cite>{{"{{"}} .Name {{"}}}}</cite>, <time>{{"{{"}} .Timestamp {{"}}}}</time></p>
        </li>
        {{"{{"}} end {{"}}}}
    </ul>
{% endcodeblock %}

Notice we don't worry about any kind of injection. The `html/template` package
is super awesome and handles that by knowing what it's outputing and the context
in which the data is being used. If you're in an html context, it will escape
the html properly. If you're in a script or url context, it knows and will apply
the appropriate esacping. No modifying the data in the database. No "sanitizing".
Just doing the right thing, every time.

## Signing it

Time to add the handler to sign the guest book. Let's start with the html for
the form.

{% codeblock templates/index.html %}
    <hr>

    <form action="/sign" method="POST">
        <p>Name: <input type="text" name="name"></p>
        <p>Message: <textarea name="message" rows="10" cols="40"></textarea></p>
        <p><button>Sign</button></p>
    </form>
{% endcodeblock %}

And now the handler:

{% codeblock sign.go %}
package main

import "net/http"

func sign(w http.ResponseWriter, req *http.Request) {
	//make sure we got post
	if req.Method != "POST" {
		http.NotFound(w, req)
		return
	}

	entry := NewEntry()
	entry.Name = req.FormValue("name")
	entry.Message = req.FormValue("message")

	if entry.Name == "" {
		entry.Name = "Some dummy who forgot a name"
	}
	if entry.Message == "" {
		entry.Message = "Some dummy who forgot a message."
	}

	s := session.Clone()
	defer s.Close()

	coll := s.DB("gostbook").C("entries")
	if err := coll.Insert(entry); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	http.Redirect(w, req, "/", http.StatusTemporaryRedirect)
}
{% endcodeblock %}

All we need to do is add a single line to main.go to make it handle the new
handler:

	http.HandleFunc("/sign", sign)

And we can sign, and view our guestbook. Lets commit again.

## Some issues

Now the astute reader will notice a couple little pain points.

- We had to check in the `sign` handler if the method was `POST`. This can
be fixed by using a more sophisticated muxer than the built in one in `net/http`.
Like all good packages in Go, all of these things are just interfaces and so you
can swap them out with many community driven packages. An exellent one is the
gorilla muxer at `code.google.com/p/gorilla/mux`.

- We had to hard code the urls. Once again, this is solved by using a more
sophisticated muxer. `code.google.com/p/gorilla/mux` supports building urls
from names you give to the routes.

- Boilerplate in the handlers to specify a database/collection every time.
I typically solve this how I wrote earlier by making a type that implements
the `ServeHTTP` method and passes in a request context containing everything
I need to use for that request, including sessions and database connections.
It's only a couple lines of code to make, but outside the scope of this
post.

Other than that, I found it to be pretty painless and about as easy to do as
the Flask version. Considering this is a statically typed compiled language,
that's quite the feat.

## Deployment

It wouldn't be useful if it wasn't deployed. Fortunately, Go compiles down into
a static binary. This can be shipped to any system that it was compiled for, and
just ran. Go also allows you to easily cross compile for any system, so thats a
non-issue as well. The built in web server is comparable in performance to things
like Apache and nginx from my tests. So for most cases, it's as simple as running
a binary and either proxy passing it through from your front end server, or just
letting the world hit it directly.

But, since that's not cool enough, we're also going to deploy on Heroku.

## Buildpacks and a Note About Getting Code

Unfortunately, Go isn't a supported platform on Heroku. Fortunately, it's just
a buildpack away. The Cedar stack is excellent and allows you to run any binary
you want to host your web site, so we just have to tell Heroku how to build our
code. I'm a little biased so I'm going to use the buildpack I created to do this,
although there are alternatives.

The cool part about hosting our code on github is that anyone with Go installed
can just grab it with a single command:

	go get github.com/zeebo/gostbook

That will download, compile, and install a binary named "gostbook" in our bin
directory in our GOPATH. The buildpack I created uses this to build the code
we'll be deploying. First we make a little file that describes how to do it,
and a Procfile to describe what to run:

{% codeblock .heroku %}
BASE=github.com/zeebo/gostbook
+ github.com/zeebo/gostbook
{% endcodeblock %}

{% codeblock Procfile %}
web: bin/gostbook
{% endcodeblock %}

Then we have to be nice and listen on the port Heroku tells us to. This is
a one line change:

	if err = http.ListenAndServe(":"+os.Getenv("PORT"), nil); err != nil {

Lastly, we have to dail out to the mongo config they ask too:

	session, err = mgo.Dial(os.Getenv("DATABASE_URL"))

I use `DATABASE_URL` as the key. We'll have to set it later in the deployment.
Let's commit that.

## Deployment (again)

Lets create the heroku app.

	sigma:gostbook zeebo$ heroku create --stack cedar --buildpack http://github.com/zeebo/buildpack.git
	Creating tranquil-refuge-9104... done, stack is cedar
	http://tranquil-refuge-9104.herokuapp.com/ | git@heroku.com:tranquil-refuge-9104.git
	Git remote heroku added

Add in a free mongo database and configure the `DATABASE_URL`:

	sigma:gostbook zeebo$ heroku addons:add mongolab:starter
	-----> Adding mongolab:starter to tranquil-refuge-9104... done, v3 (free)
	       Welcome to MongoLab.
	sigma:gostbook zeebo$ heroku config
	BUILDPACK_URL => http://github.com/zeebo/buildpack.git
	MONGOLAB_URI  => ...snip...
	sigma:gostbook zeebo$ heroku config:add DATABASE_URL=...snip...
	Adding config vars and restarting app... done, v4
	  DATABASE_URL => ...snip...

If I was smarter, I would have just used `MONGOLAB_URI` in the code, but I'm not
so here we are. Finally, we can just push it up and watch the magic:

	sigma:gostbook zeebo$ git push heroku master
	Counting objects: 24, done.
	Delta compression using up to 4 threads.
	Compressing objects: 100% (21/21), done.
	Writing objects: 100% (24/24), 3.41 KiB, done.
	Total 24 (delta 4), reused 0 (delta 0)

	-----> Heroku receiving push
	-----> Fetching custom buildpack... done
	-----> Go app detected
	-----> Configuration
	       GO_VERSION=go1.0.2
	       BASE=github.com/zeebo/gostbook
	       + github.com/zeebo/gostbook
	-----> Using Go go1.0.2.linux-amd64
	-----> Fetching Go go1.0.2.linux-amd64
	-----> Checking for Mercurial and Bazaar
	       Fetching hg and bzr
	       ..snip...
	       Successfully installed mercurial
	       ...snip...
	       Successfully installed bzr
	       Cleaning up...
	-----> Running go get -u -v all
	-----> Copying sources into GOPATH/src/github.com/zeebo/gostbook
	-----> Running go get -v github.com/zeebo/gostbook
	       Fetching https://labix.org/v2/mgo?go-get=1
	       Parsing meta tags from https://labix.org/v2/mgo?go-get=1 (status code 200)
	       get "labix.org/v2/mgo": found meta tag main.metaImport{Prefix:"labix.org/v2/mgo", VCS:"bzr", RepoRoot:"https://launchpad.net/mgo/v2"} at https://labix.org/v2/mgo?go-get=1
	       labix.org/v2/mgo (download)
	       Fetching https://labix.org/v2/mgo/bson?go-get=1
	       Parsing meta tags from https://labix.org/v2/mgo/bson?go-get=1 (status code 200)
	       get "labix.org/v2/mgo/bson": found meta tag main.metaImport{Prefix:"labix.org/v2/mgo", VCS:"bzr", RepoRoot:"https://launchpad.net/mgo/v2"} at https://labix.org/v2/mgo/bson?go-get=1
	       get "labix.org/v2/mgo/bson": verifying non-authoritative meta tag
	       Fetching https://labix.org/v2/mgo?go-get=1
	       Parsing meta tags from https://labix.org/v2/mgo?go-get=1 (status code 200)
	       labix.org/v2/mgo/bson
	       labix.org/v2/mgo
	       github.com/zeebo/gostbook
	-----> Discovering process types
	       Procfile declares types -> web
	-----> Compiled slug size is 1.4MB
	-----> Launching... done, v6
	       http://tranquil-refuge-9104.herokuapp.com deployed to Heroku

	To git@heroku.com:tranquil-refuge-9104.git
	 * [new branch]      master -> master

And we have a nice guestbook at http://tranquil-refuge-9104.herokuapp.com

## A snag

It seems like the database name is specified by the host in this case. We can't
just go and create whatever database we want. So we have to update the code to
grab this information and use it when we're making queries. The patch to fix it
was pretty easy. Just add a global variable and parse the URL to put the database
into it.

{% codeblock commit.diff %}
diff --git a/main.go b/main.go
index 6094df8..eea1565 100644
--- a/main.go
+++ b/main.go
@@ -4,6 +4,7 @@ import (
 	"html/template"
 	"labix.org/v2/mgo"
 	"net/http"
+	"net/url"
 	"os"
 )
 
@@ -19,7 +20,7 @@ func hello(w http.ResponseWriter, req *http.Request) {
 	defer s.Close()
 
 	//set up the collection and query
-	coll := s.DB("gostbook").C("entries")
+	coll := s.DB(database).C("entries")
 	query := coll.Find(nil).Sort("-timestamp")
 
 	//execute the query
@@ -38,10 +39,17 @@ func hello(w http.ResponseWriter, req *http.Request) {
 }
 
 var session *mgo.Session
+var database string
 
 func main() {
 	var err error
-	session, err = mgo.Dial(os.Getenv("DATABASE_URL"))
+	u := os.Getenv("DATABASE_URL")
+	parsed, err := url.Parse(u)
+	if err != nil {
+		panic(err)
+	}
+	database = parsed.Path[1:]
+	session, err = mgo.Dial(u)
 	if err != nil {
 		panic(err)
 	}
diff --git a/sign.go b/sign.go
index a5b6cd0..c3ddbda 100644
--- a/sign.go
+++ b/sign.go
@@ -23,7 +23,7 @@ func sign(w http.ResponseWriter, req *http.Request) {
 	s := session.Clone()
 	defer s.Close()
 
-	coll := s.DB("gostbook").C("entries")
+	coll := s.DB(database).C("entries")
 	if err := coll.Insert(entry); err != nil {
 		http.Error(w, err.Error(), http.StatusInternalServerError)
 		return
{% endcodeblock %}

We just rely on the `net/url` package to parse the url and grab the database
out of the path argument. Since the path contains the leading forward slash, we
just slice that off. All thats left is a redeploy:

	sigma:gostbook zeebo$ git add .
	sigma:gostbook zeebo$ git commit -m 'fixes for database'
	[master 2b4bf78] fixes for database
	 2 files changed, 11 insertions(+), 3 deletions(-)
	sigma:gostbook zeebo$ git push heroku master
	Counting objects: 7, done.
	Delta compression using up to 4 threads.
	Compressing objects: 100% (4/4), done.
	Writing objects: 100% (4/4), 493 bytes, done.
	Total 4 (delta 3), reused 0 (delta 0)

	-----> Heroku receiving push
	-----> Fetching custom buildpack... done
	-----> Go app detected
	-----> Configuration
	       GO_VERSION=go1.0.2
	       BASE=github.com/zeebo/gostbook
	       + github.com/zeebo/gostbook
	-----> Using Go go1.0.2.linux-amd64
	-----> Checking for Mercurial and Bazaar
	       /app/tmp/repo.git/.cache/venv/bin/hg
	       /app/tmp/repo.git/.cache/venv/bin/bzr
	-----> Running go get -u -v all
	-----> Copying sources into GOPATH/src/github.com/zeebo/gostbook
	-----> Running go get -v github.com/zeebo/gostbook
	       github.com/zeebo/gostbook
	-----> Discovering process types
	       Procfile declares types -> web
	-----> Compiled slug size is 1.4MB
	-----> Launching... done, v7
	       http://tranquil-refuge-9104.herokuapp.com deployed to Heroku

	To git@heroku.com:tranquil-refuge-9104.git
	   52a2171..2b4bf78  master -> master

And to my surprise, it worked on the second try!

## Closing remarks

Dunno. Welp. Go is neat.