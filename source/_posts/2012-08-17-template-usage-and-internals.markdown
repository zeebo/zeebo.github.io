---
layout: post
title: "Template Usage and Internals"
date: 2012-08-17 11:51
comments: true
categories: [go, templates]
---

I've seen many tutorials on the internet about how to use templates in Go. They
typically concentrate on the syntax of the template and don't go in to the details
of how they're constructed and used in Go code. That's why this article is about
what a template really is, and how to use them in your code.

<!-- more -->

## A little history

Back before Go 1, the [text/template][text/template] package was different. I think it was just
named template, but the real major difference was there were two types. A template
and a template set. At some point the library authors decided to merge them into
one, the [Template][text/template.Template] type.

## So what is a template?

I like to think of a template as a collection of templates, with one promoted
as the "default" template. The default template is the one that gets used when 
the execute method is called. Interally, a template contains a map of all the
other templates it is linked with, and each one of those templates contains
the same map. This makes the namespace of a template flat. It sounds really
confusing, so hopefully we can make it easier with some examples and implications.

## Implications

So given that a template is really a set of templates with a mapping of all the
template names in the set to the templates (phew), what can we expect in how
we work with it?

- __Every template in the set can be called by any other.__ Because every template
shares the same map of name to templates, any template can call any other templates.
This is a pretty simple.

- __You can't have two templates with the same name.__ This is also pretty obvious
but bears stating. Because the namespace is flat, two template with the same name
would cause a collision. The package doesn't allow you to add a template 

- __Theres no such thing as a subtemplate.__ Because the namespace is flat, you can't
really have one template be a subtemplate of another. Because every template is accessable
from every other template, the concept of one template owning another isn't really
defined. This isn't to say you can't use templates like subtemplates, its just that
theres no mechanism encforcing this in the package.

- __Lookup is idempotent.__ This means that calling `t.Lookup("name").Lookup("name")`
is the same as `t.Lookup("name")`, as long as `"name"` exists in the template. This
is because every template shares the same map of template name to template. When we
lookup a specific one, it still has the same map to lookup the next one.

## Gotchas

Here's some common problems with the template package that people run into:

- __ParseFiles and it's friends add the template under the name of the file.__ This means
if you do `template.New("base").ParseFiles("foo.html")` and try to execute it, you will
have an empty template. The template was read and parsed from "foo.html" and added to the
template under that name. This means you have to either do a `.Lookup("foo.html")` or
change the name in New to "foo.html".

- __Functions must be added before parsing.__ The template package has to know what
the names of the functions you're going to call during parse time in order to parse
it correctly. This can make the code a little longer, especailly when you start with
a simple `template.ParseFiles` and need to add functions to it.

## Quiz time

Now that we know more about how templates work and some common gotchas, lets look
at some code and have a quiz. Here's how the quiz works: I'll show you a piece
of sample code consisting of some files, and you tell me the output.
The answers are at the bottom of this post.

### Question 1

{% codeblock main.go %}
package main

import (
	"text/template"
	"os"
)

func main() {
	t := template.Must(template.New("foo").Parse(`{{"{{"}} {{"}}}}`))
	t.Execute(os.Stdout, nil)
}
{% endcodeblock %}

[text/template]: http://golang.org/pkg/text/template
[text/template.Template]: http://golang.org/pkg/text/template/#Template