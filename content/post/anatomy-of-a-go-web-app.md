+++
title = "Anatomy of a Go Web Application"
draft = true
date = "2016-11-07T16:53:07-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference"]

+++

When building a web application from scratch, there are a lot of decisions to make.  The goal of this guide is to give 
one more example of how you can go about building a web application in the Go language, as well as to give you and idea 
what things you need to start thinking about and plan for before you get started.

This guide is not intended to exhaustive, nor is it absolute. It is a compendium of the things I thought about and how I dealt 
with them when building [townsourced.com](https://www.townsourced.com).  Hopefully you'll find some of it useful.

# Package Layout and Code Organization

Clean separation of duties, and clear code boundaries are crucial to figure out when building large applications.  I've
found, with web applications, these the 3 package delineations are required:

* [web](#the-web-package)
* [app](#the-app-package)
* [data](#the-data-package)

There may be others, but for the most part your code will fall into one of these 3 packages, or sub-packages 
within them.

But before we get into packages, we need to setup the main entry point to our program.  In Go, this means `func main()`
and I usually put mine in `main.go`.

## main.go

`main.go` will be responsible for loading anything configurable in your web app and passing those configuration options
into the other layers.  There are many different options for storing your app's configuration, personally I store mine 
in json files using [small configuration library](https://github.com/timshannon/config) I wrote.  If you want many options
including environment variables, TOML, YAML, or JSON , check out  Steve Francia's excellent [Viper library](https://github.com/spf13/viper).

Below is an example of townsourced's config file to give you an idea of the types of options you may put in there:

```javascript
{
    "app": {
        "httpClientTimeout": "30s", 
        "taskPollTime": "1m",
        "taskQueueSize": 100
    },
    "data": {
        "cache": { // memcached servers
            "addresses": [
                "127.0.0.1:11211"
            ]
        },
        "db": { // RethinkDB 
            "address": "127.0.0.1:28015",
            "database": "townsourced",
            "timeout": "60s"
        },
        "search": {  // ElasticSearch
            "addresses": [
                "http://127.0.0.1:9200"
            ],
            "index": {
                "name": "townsourced",
                "replicas": 1,
                "shards": 5
            },
            "maxRetries": 0
        }
    },
    "web": {
        "address": "http://localhost:8080",
        "certFile": "", 
        "keyFile": "",
        "maxHeaderBytes": 0,
        "maxUploadMemoryMB": 10,
        "minTLSVersion": 769,
        "readTimeout": "60s",
        "writeTimeout": "60s"
    }
}
```

This configuration will usually be defined as separate structs in each package: `web.Config`, `app.Config`, and `data.Config`.
You'll then load the configuration into these structs, and pass them into their respective packages. This is also a good 
time to initialize any clients in those packages as well.

## The Web Package

The **web** package is the client's entry point to your application.  Here you will extensively use the [http package](https://golang.org/pkg/net/http/) 
and the [encoding/json package](https://golang.org/pkg/encoding/json") for any REST APIs.

You will process cookies at this level and pass already extracted session identifies to your application layer.  The http
package and it's types and interfaces (requests, writers, cookies, etc.) should never leave this package.  Or to put it
another way you shouldn't need to import http in *any* other packages except this one.  Preventing http from leaking into
other packages will help keep clear lines of responsibility between your packages.

## The App Package

The **app** package is where the meat of your application will be.  Inside this package you'll define exported types
that will be passed to and from your other packages; types like `app.User` and `app.Session`.  The point of the app package
is to validate and handle data within these types, so most of your code in the app package will be methods on types, or
functions that return these types. Be conservative about which types and methods you choose to export.  The web package
doesn't need to understand how an `app.User` is approved to see data, it just needs to know if they can or not.

More likely than not your application data will be related to each other.  Try to retrieve types from their relationship
with other types rather than from stand alone functions.

```Go
	//  instead of this
	func UserFromSession(s *Session) (*User, error) {

	}

	// do this
	func(s *Session) User() (*User, error){

	}

```

If standalone functions are necessary or make more sense (for instance when they don't actually modify any types), then 
don't export them.

If you aren't careful you may find yourself implementing parts of your application in the web or data layers when they
should be in the app layer.  For example, access controls.

You may be tempted to check if a user has access at the web layer, but this type of access control should always be done
at the app layer, where it can be easily unit tested.  

```Go
// instead of this

package web

func handler(w http.ResponseWriter, r *http.Request) {
	if currentUser.Equal(userBeingViewed) {
		profile, err := userBeingViewed.Profile()
	}
}

// do this

package app

func (u *User) Profile(who *User) (*ProfileData, error) {
	if !u.equal(who) {
		return nil, errors.New("You do not have access to view this profile!")
	}

	return &ProfileData{}, nil
}


```

## The Data Package

The data package is for getting and setting data in your persistent or temporary storage.  There shouldn't be any types
kept here unless they are types that augment the data layer. 


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

