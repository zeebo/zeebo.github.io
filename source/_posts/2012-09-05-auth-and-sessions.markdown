---
layout: post
title: "Auth and Sessions"
date: 2012-09-05 16:10
comments: true
categories: [go, golang, web, tutorial, sessions, auth, mgo, mongodb]
---

I think it's time to bring the guestbook to the next level, and that means users
and sessions. This post will show you how to handle user registration and authentication.
Let's get started!

<!-- more -->

## The User Type

The first thing we're going to do is create a type to store the information of
the user. So what do users have? Well, an ID to identify them in the database,
a username, and a password. To make this a little more fun, we're also going to
store the number of times they've posted on the guestbook. So here's our type:

{% codeblock user.go %}
package main

import (
	"code.google.com/p/go.crypto/bcrypt"
	"labix.org/v2/mgo/bson"
)

type User struct {
	ID       bson.ObjectId `bson:"_id,omitempty"`
	Username string
	Password []byte
	Posts    int
}
{% endcodeblock %}

Now lets define some functions to help hash the password and set it on the user
and and authenticate a user given a username and password.

{% codeblock user.go %}
//SetPassword takes a plaintext password and hashes it with bcrypt and sets the
//password field to the hash.
func (u *User) SetPassword(password string) {
	hpass, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		panic(err) //this is a panic because bcrypt errors on invalid costs
	}
	u.Password = hpass
}

//Login validates and returns a user object if they exist in the database.
func Login(ctx *Context, username, password string) (u *User, err error) {
	err = ctx.C("users").Find(bson.M{"username": username}).One(&u)
	if err != nil {
		return
	}

	err = bcrypt.CompareHashAndPassword(u.Password, []byte(password))
	if err != nil {
		u = nil
	}
	return
}
{% endcodeblock %}

Now lets work on the handler to log them in.

## First Sign of Trouble

The login handler should be pretty simple. All we have to do is get the username
and password from the form POSTed to the handler, and pass it to our Login function
which will grab the user from the database and authenticate the credentials.

{% codeblock handlers.go %}
func login(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	//grab the username and password from the form
	username, password := req.FormValue("username"), req.FormValue("password")

	//log in the user
	user, err := Login(ctx, username, password)

	//what to do now? if there was an error we want to present the form again
	//with some error message.

	//where do we store the user if the login was valid?

	//answer: sessions!
	_ = user
	return
}
{% endcodeblock %}

But we ran into some trouble. When should we display the template for the form?
Where do we store that the authentication was correct? Fortunately it's not too
hard to fix these problems. Lets handle the displaying of the template part first.

## Two Handlers Are Better Than One

The login handler is really two actions. When a GET request is passed to the handler
it should display a nice form, but when a POST request is passed to the handler
it should authenticate a user. These different actions based on the verb used
on the URL means we should dispatch to the correct handler in the router rather
than the handler itself. Lets write the simple form displaying template first.

{% codeblock handlers.go %}
var login = parseTemplate(
	"templates/_base.html",
	"templates/login.html",
)

func loginForm(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	err = login.Execute(w, nil)
	return
}
{% endcodeblock %}

The code was getting "smelly" because it wasn't nice to have global variables storing
the templates, so it wasn't too hard to whip up a simple function to compile templates
and cache them on the fly. Here's what that looks like, and the new handler using it:

{% codeblock template.go %}
package main

import (
	"html/template"
	"path/filepath"
	"sync"
)

var cachedTemplates = map[string]*template.Template{}
var cachedMutex sync.Mutex

var funcs = template.FuncMap{
	"reverse": reverse,
}

func T(name string) *template.Template {
	cachedMutex.Lock()
	defer cachedMutex.Unlock()

	if t, ok := cachedTemplates[name]; ok {
		return t
	}

	t := template.New("_base.html").Funcs(funcs)

	t = template.Must(t.ParseFiles(
		"templates/_base.html",
		filepath.Join("templates", name),
	))
	cachedTemplates[name] = t

	return t
}
{% endcodeblock %}

{% codeblock handlers.go %}
func loginForm(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	return T("login.html").Execute(w, nil)
}
{% endcodeblock %}

{% codeblock Login Template %}
{{"{{"}} define "title" {{"}}}}Guestbook - Login{{"{{"}} end {{"}}}}

{{"{{"}} define "content" {{"}}}}
    <h1>Login</h1>
    <form action="{{"{{"}} reverse "login" {{"}}}}" method="POST">
        <p>Username: <input type="text" name="username"></p>
        <p>Password: <input type="password" name="password"></p>
        <p><button>Login</button></p>
    </form>
{{"{{"}} end {{"}}}}
{% endcodeblock %}

At this point I had a working login page that would 404 when you clicked Login.
There were some smaller changes made around to clean things up that you can see
in [this commit][gostbook:handlers] (I have annotated the commit to include
some comments on the changes.) Let's add the login form handling now.

## Sessions

A session is just some data attached to some id that you hand the client in a cookie.
This way when a client requests a page, you can look at the cookie value and get the
id for the data and load up the data for that request. Tada! Sessions! For our
implementation of sessions, we're once again going to use the excellent
[gorilla][gorilla/sessions] package for sessions. It lets you use different stores
for the backend data, and in this case we're just going to use a [cookie store][cookiestore].
This stores all the data in the cookie the client sends to you. This does mean
that the user can tamper with the cookie, but the data is verified using a secret
value and a hash, and can optionally be encrypted with another secret value. For this
I'm just going to use a store that doesn't encrypt the data: after all, the data
the store uses is open source.

{% codeblock main.go %}
import (
	"code.google.com/p/gorilla/sessions"
	//...
)

var store sessions.Store
//...


func main() {
	//...
	store = sessions.NewCookieStore([]byte(os.Getenv("KEY")))
}
{% endcodeblock %}

So we defined a cookie store, now let's add grabbing the session to the context.

{% codeblock context.go %}
type Context struct {
	Database *mgo.Database
	Session  *sessions.Session
}

func NewContext(req *http.Request) (*Context, error) {
	sess, err := store.Get(req, "gostbook")
	return &Context{
		Database: session.Clone().DB(database),
		Session:  sess,
	}, err
}
{% endcodeblock %}

The last thing we need to do is make sure the handlers save the session when they're
done with it. Unfortunately this causes a problem. Saving the session requires 
modifying the headers of the response, and if the handler has already started
outputting data, the headers have already been sent and that ship has sailed.
Theres two approaches to solving this problem. The first is to just make sure
in each handler to save the session before writing anything to the `ResponseWriter`,
which can be a little verbose and error prone but provides the best performance.
The second is to use the fact that a `ResponseWriter` is an interface and use our
handler type to substitute in a buffered `ResponseWriter` that stores all the data
and header information written to it, so that it can be output at the end all at once.
I wrote a package to help with the second option so it's clearly the one I prefer.
Here's how we can hook that up:

{% codeblock http.go %}
package main

import (
	"net/http"
	"thegoods.biz/httpbuf"
)

type handler func(http.ResponseWriter, *http.Request, *Context) error

func (h handler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	//create the context
	ctx, err := NewContext(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
	defer ctx.Close()

	//run the handler and grab the error, and report it
	buf := new(httpbuf.Buffer)
	err = h(buf, req, ctx)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}

	//save the session
	if err = ctx.Session.Save(req, buf); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}

	//apply the buffered response to the writer
	buf.Apply(w)
}
{% endcodeblock %}

All we do is create an `httpbuf.Buffer` and use that as our handler, finishing
with a call to its Apply method. With that, we can set and grab session values
in the handlers by just interacting with `ctx.Session`, and everthing will be
saved when [we're done][gostbook:sessions].

## Back to Authentication

Now that we have sessions, we know where we can store the user. Lets write the
login handler for the user then.

{% codeblock handlers.go %}
func login(w http.ResponseWriter, req *http.Request, ctx *Context) error {
	username, password := req.FormValue("username"), req.FormValue("password")

	user, e := Login(ctx, username, password)
	if e != nil {
		ctx.Session.AddFlash("Invalid Username/Password")
		return loginForm(w, req, ctx)
	}

	//store the user id in the values and redirect to index
	ctx.Session.Values["user"] = user.ID
	http.Redirect(w, req, reverse("index"), http.StatusSeeOther)
	return nil
}
{% endcodeblock %}

{% codeblock main.go %}
import (
	"encoding/gob"
	"labix.org/v2/mgo/bson"
	//...
)

func init() {
	gob.Register(bson.ObjectId(""))
}

func main() {
	//...
	router.Add("POST", "/login", handler(login))
	//...
}
{% endcodeblock %}

Note that we have to register the `bson.ObjectId` type with the [gob package][encoding/gob]
because the cookie store uses gob to store the data for the session.

Well, now we log in people and store it in the session, but it'd be nice if that
was reflected somehow in the user interface and if the context included information
about the logged in user. Lets do some work on the context and handlers to fix this.
First, we're going to add a `*User` to the context that gets filled in based on the
id we stored in the session.

{% codeblock context.go %}
type Context struct {
	Database *mgo.Database
	Session  *sessions.Session
	User     *User
}

func NewContext(req *http.Request) (*Context, error) {
	sess, err := store.Get(req, "gostbook")
	ctx := &Context{
		Database: session.Clone().DB(database),
		Session:  sess,
	}
	if err != nil {
		return ctx, err
	}

	//try to fill in the user from the session
	if uid, ok := sess.Values["user"].(bson.ObjectId); ok {
		err = ctx.C("users").Find(bson.M{"_id": uid}).One(&ctx.User)
	}

	return ctx, err
}
{% endcodeblock %}

Now we just have to add the context to the value we pass in to templates to be
executed and hook up the templates. [This commit][gostbook:users] shows the
details of that, including adding a logout handler, and fixing some minor issues
with the code.

## Post Count and Creating Users

The last two features we need are letting people register and increasing a persons
post count when they post. Let's work on registration first. Registration works
just like logging in, so we need to create a template and two handlers.

{% codeblock handlers.go %}
func registerForm(w http.ResponseWriter, req *http.Request, ctx *Context) (err error) {
	return T("register.html").Execute(w, map[string]interface{}{
		"ctx": ctx,
	})
}

func register(w http.ResponseWriter, req *http.Request, ctx *Context) error {
	username, password := req.FormValue("username"), req.FormValue("password")

	u := &User{
		Username: username,
		ID:       bson.NewObjectId(),
	}
	u.SetPassword(password)

	if err := ctx.C("users").Insert(u); err != nil {
		ctx.Session.AddFlash("Problem registering user.")
		return registerForm(w, req, ctx)
	}

	//store the user id in the values and redirect to index
	ctx.Session.Values["user"] = u.ID
	http.Redirect(w, req, reverse("index"), http.StatusSeeOther)
	return nil
}
{% endcodeblock %}

The `register.html` template is very similar to the login template. If you really
wan't to see it, you can find it at [this commit][gostbook:register]. Incrementing
the post count is super simple. In the `sign` handler, we just add

	ctx.C("users").Update(bson.M{"_id": ctx.User.ID}, bson.M{
		"$inc": bson.M{"posts": 1},
	})

## Phew!

So after all that we have a login/user registration system, and a session tied
to a context for storing whatever data we want. Hopefully with this guide you can
extend it to meet the needs of whatever webapp you're writing. Thanks for reading
so far, and be sure to [register and leave a comment][gostbook] on the gostbook or
right below if you liked it.

[gostbook:handlers]: https://github.com/zeebo/gostbook/commit/50d85a72495ac676db9301df45f84ca1cbf96702
[gostbook:sessions]: https://github.com/zeebo/gostbook/commit/85b30a6efb917ab8567d3d277d5373b3e12a7f67
[gostbook:users]: https://github.com/zeebo/gostbook/commit/7c2ff89f944d78cb51033c3ac5eeedd4ab552126
[gostbook:register]: https://github.com/zeebo/gostbook/commit/46bf28e9f862d083e6a40d813ff3f271bf96a5f1
[gostbook]: http://tranquil-refuge-9104.herokuapp.com
[gorilla/sessions]: http://gorilla-web.appspot.com/pkg/sessions
[encoding/gob]: http://golang.org/pkg/encoding/gob
[cookiestore]: http://gorilla-web.appspot.com/pkg/sessions#CookieStore