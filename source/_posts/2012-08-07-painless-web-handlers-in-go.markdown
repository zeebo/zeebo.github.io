---
layout: post
title: "Painless Web Handlers in Go"
date: 2012-08-07 14:13
comments: true
categories: [go, web, tutorial]
---

[Last time][quick] we made a little guestbook application, but there was a
couple little pain points. We had to have some boiler plate at the top of
all of the handlers, and errors were handled by copying the same line of code
everywhere. We also had fixed url paths hard coded in handlers and templates.
Let's see how we can fix that.

<!-- more -->

## Adding context

A lot of the boiler plate in the handlers last time had to do with the database
for each request, so let's start by cleaning that up. How we do this is by
creating a type that will have the context for the request.

{% codeblock context.go %}
package main

import (
	"labix.org/v2/mgo"
	"net/http"
)

type Context struct {
	Database *mgo.Database
}

func (c *Context) Close() {
	c.Database.Session.Close()
}

func NewContext(req *http.Request) (*Context, error) {
	return &Context{
		Database: session.Clone().DB(database),
	}, nil
}
{% endcodeblock %}

A context is the general context the request will use to make decisions, bundled
up with the handles to the resources it needs to perform actions. Right now we
only have the database. Let's change our handlers to use the new context.

{% codeblock main.go %}
func hello(w http.ResponseWriter, req *http.Request) {
	ctx, err := NewContext(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer ctx.Close()

	//set up the collection and query
	coll := ctx.Database.C("entries")
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

	ctx, err := NewContext(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer ctx.Close()

	coll := ctx.Database.C("entries")
	if err := coll.Insert(entry); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	http.Redirect(w, req, "/", http.StatusTemporaryRedirect)
}
{% endcodeblock %}

Now thats wonderful, but it looks like we just made it worse.

## The magic of interfaces

To fix this, we're going to create a new handler type, and give it a 
`ServeHTTP` method. This new handler type will handle creating/closing
the context, and handling any errors that arise. Here's the definition:

{% codeblock http.go %}
package main

import "net/http"

type handler func(http.ResponseWriter, *http.Request, *Context) error

func (h handler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	//create the context
	ctx, err := NewContext(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
	defer ctx.Close()

	//run the handler and grab the error, and report it
	err = h(w, req, ctx)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
{% endcodeblock %}

The `handler` type is a function type, meaning any function with that signature
can be cast to that type. We define a method on the function (I know!) so that
the net/http package can use it as though it were any other handler. We've
already been doing something very similar to this already. When we called the 
`http.HandleFunc` function in our `main.go`, we've been using our functions
as the type `http.HandlerFunc` which defines a ServeHTTP method, just like ours.
See, it's not so bad. Here's what the new handlers look like:

{% codeblock handlers.go %}
package main

import "net/http"

func hello(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	//set up the collection and query
	coll := ctx.Database.C("entries")
	query := coll.Find(nil).Sort("-timestamp")

	//execute the query
	//TODO: add pagination :)
	var entries []Entry
	if err = query.All(&entries); err != nil {
		return
	}

	//execute the template
	err = index.Execute(w, entries)
	return
}

func sign(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
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

	coll := ctx.Database.C("entries")
	if err = coll.Insert(entry); err != nil {
		return
	}

	http.Redirect(w, req, "/", http.StatusTemporaryRedirect)
	return
}
{% endcodeblock %}

Much better! Let's commit that.

## Routing

The other pain points, hard coded urls, and checking the request method,
are going to be handled by more advanced routing. For this, we're going
to use the execllent [gorilla web toolkit][gorilla], specifically the
[gorilla/pat][gorilla/pat] package. I really like the simple API it provides
with easy parameter capturing from the url. It's very easy to use with the
`net/http` package:

{% codeblock main.go %}
func main() {
	var err error
	u := os.Getenv("DATABASE_URL")
	parsed, err := url.Parse(u)
	if err != nil {
		panic(err)
	}
	database = parsed.Path[1:]
	session, err = mgo.Dial(u)
	if err != nil {
		panic(err)
	}

	r := pat.New()
	r.Add("GET", "/", handler(hello)).Name("index")
	r.Add("POST", "/sign", handler(sign)).Name("sign")

	if err = http.ListenAndServe(":"+os.Getenv("PORT"), r); err != nil {
		panic(err)
	}
}
{% endcodeblock %}

One important and easy to miss detail is we now pass the router in as the
second argument to the `http.ListenAndServe` call. Now we can remove the check
that the method is `POST` in the `sign` handler, as the router takes care of that
for us. Lets move on to fixing the hard coded entries.

## Reversing URLs

If you'll notice, we gave the handlers a `.Name` call. The [gorilla/pat][gorilla/pat]
package returns a `*mux.Router` for us to work with. Using that we can have the
router rebuild urls from the names. For example, if we wanted to grab the url for
the index page, we could use

	r.GetRoute("index").URL()

but since `r` is inaccessable outside the `main` function, we have to move it
into a higher scope. Let's do that.

{% codeblock main.go %}
var router *pat.Router

func main() {
	//...

	router = pat.New()
	router.Add("GET", "/", handler(hello)).Name("index")
	router.Add("POST", "/sign", handler(sign)).Name("sign")

	if err = http.ListenAndServe(":"+os.Getenv("PORT"), router); err != nil {
		panic(err)
	}
}
{% endcodeblock %}

And now we can update the sign handler

{% codeblock handlers.go %}
func sign(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	//...

	url, err := router.GetRoute("index").URL()
	if err != nil {
		return
	}

	http.Redirect(w, req, url, http.StatusTemporaryRedirect)
	return
}
{% endcodeblock %}

## Reversing in Templates

To reverse inside the template, we could either remember to pass the router in
as part of the template context on every invocation, or we could add a function to
the template. Since keeping track of the router through nested templates and scope
changes is a daunting task, adding a function to do the reversing is a better
option. Heres that function:

{% codeblock main.go %}
func reverse(name string, things ...interface{}) string {
	//convert the things to strings
	strs := make([]string, len(things))
	for i, th := range things {
		strs[i] = fmt.Sprint(th)
	}
	//grab the route
	u, err := router.GetRoute(name).URL(strs...)
	if err != nil {
		panic(err)
	}
	return u.Path
}
{% endcodeblock %}

We choose to have the function panic on errors because any incorrect reversal
is a programmer error. We also accept a variadic number of interface values because
sometimes we need to have a parameter in the reversal that is an integer, like the
year on the blog post url, and the URL function takes strings. So rather than force
the template to do the conversion, or the function executing the template, we
just convert everything to a string by calling `fmt.Sprint` on it. Then we have
to add this function to the template.

{% codeblock main.go %}
var funcs = template.FuncMap{
	"reverse": reverse,
}

func parseTemplate(files ...string) *template.Template {
	//create a new template named after the first file in the list and add
	//the function map to it
	t := template.New(files[0]).Funcs(funcs)

	//parse the files into the template and panic on errors
	t = template.Must(t.ParseFiles(files...))
	return t
}

var index = parseTemplate(
	"templates/_base.html",
	"templates/index.html",
)
{% endcodeblock %}

Theres a tricky point here: the template package will error when trying to parse
a template and it finds a function invocation to something undefined. That means
we have to add our function map to the template before we add the files to parse.
We write a little helper function to do this correctly. Now we can update the
template to use it.

	<form action="{{"{{"}} reverse "sign" {{"}}}}" method="POST">

Let's update the `sign` handler to use the reverse function too.

	http.Redirect(w, req, reverse("index"), http.StatusSeeOther)

Pain: consider yourself eliminated.

Next up, we're going to do more with the context type we created, and make
the guestbook a little more web 2.0. As always, the source to the gostbook
is up on [github][gostbook].

[quick]: http://shadynasty.biz/blog/2012/07/30/quick-and-clean-in-go/
[gorilla]: http://gorilla-web.appspot.com
[gorilla/pat]: http://gorilla-web.appspot.com/pkg/pat
[gostbook]: http://github.com/zeebo/gostbook