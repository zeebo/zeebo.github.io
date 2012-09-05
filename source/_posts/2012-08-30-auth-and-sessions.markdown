---
layout: post
title: "Auth and Sessions"
date: 2012-08-30 13:10
comments: true
categories: [go, web, tutorial, sessions, auth, mgo, mongodb]
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

import "labix.org/v2/mgo"

type User struct {
	ID bson.ObjectId `bson:"_id,omitempty"`
}
{% endcodeblock %}