+++
title = "Anatomy of a Go Web Application"
draft = true
date = "2016-11-07T16:53:07-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference"]

+++

When building a web application from scratch, there are a lot of decisions to make.  The goal of this post is to give 
one more example of how you can go about building a web application in the Go language, as well as to give you and idea 
what things you need to start thinking about and plan for before you get started.

This guide is not intended to exhaustive nor absolute. It is a compendium of the things I thought about and how I dealt 
with them when building [townsourced.com](https://www.townsourced.com).


# Package Layout and Code Organization

Clean separation of duties, and clear code boundaries are crucial to figure out when building large applications.  I've
found, with web applications, these the 3 package delineations are required:

* web
* app
* data

There may be other packages, but for the most part your code will fall into one of these 3 packages, or sub-packages 
within them.

## The Web Package

The **web** package is the entry point to your application.  Here you will extensively use the [http package](https://golang.org/pkg/net/http/) 
and (for REST APIs) the [encoding/json package](https://golang.org/pkg/encoding/json").

You will process cookies at this level and pass already extracted session identifies to your application layer.  The http
package and it's types and interfaces (requests, writers, cookies, etc.) should never leave this package.  Or to put it
another way you shouldn't need to import http in *any* other packages except this one.  Preventing http from leaking into
other packages will help keep clear lines of responsibility between your packages.

## The App Package

The **app** package is where the meat of your application will be.  Inside this package you'll define exported types
that will be passed to and from your other packages; types like app.User and app.Session.  The point of the app package
is to validate and handle data within these types, so most of your code in the app package will be methods on types, or
functions that return these types. Be conservative about which types and methods you choose to export.  The web package
doesn't need to understand how an *app.User* is approved to see data, it just needs to know if they can or not.

More likely than not your application data will be related to each other.  Try to retrieve types from their relationship
with other types rather than from stand alone functions.

```Go
	func(s *Session) User() (*User, error){

	}

	//  instead of

	func UserFromSession(s *Session) (*User, error) {

	}
```

If stand alone functions are necessary or make more sense (for instance when they don't actually modify any types), then 
don't export them.

---

* SQL vs NOSQL
* Error Handling
 * Errors 5xx vs Failures 4xx
* Other
* JSON Handling
  * Pointers in inputs / nil input
* Templates
* Gzip data
* Tips and Tricks
 * pooling gzip and json readers and writers
 * Private data like API Keys
 * Templates and static data handling in Development environments

