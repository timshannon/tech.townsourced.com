+++
title = "Anatomy of a Go Web Application"
draft = false
date = "2016-12-04T10:37:07-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference", "Townsourced"]
keywords = ["web development", "go", "golang", "backend development", "Townsourced"]
+++

When building a web application from scratch, there are a lot of decisions to make.  The goal of this guide is to give 
one more example of how you can go about building a web application in the Go language, as well as to give you an idea 
what things you need to start thinking about and plan for before you get started.

This guide is not intended to be exhaustive, nor is it absolute. It is a compendium of the things I thought about and how I dealt 
with them when building [townsourced.com](https://www.townsourced.com).  Hopefully you'll find it useful.

<!--more-->

# Package Layout and Code Organization

Clean separation of duties, and clear code boundaries are crucial to figure out when building large applications.  I've
found, with web applications, these the 3 package delineations are required:

* [web](#the-web-package)
* [app](#the-app-package)
* [data](#the-data-package)

There may be others, but for the most part your code will fall into one of these 3 packages, or sub-packages 
within them.

But, before we get into packages, we need to setup the main entry point to our program.  In Go, this means `func main()`
and I usually put mine in `main.go`.

## main.go

`main.go` will be responsible for loading anything configurable in your web app and passing those configuration options
into the other layers.  There are many different options for storing your app's configuration, personally I store mine 
in json files using a [small configuration library](https://github.com/timshannon/config) I wrote.  If you want many options
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

I defined this configuration as separate structs in each package: `web.Config`, `app.Config`, and `data.Config`.
You'll then load the configuration into these structs, and pass them into their respective packages. This is also a good 
time to initialize any clients in those packages as well.  In Townsourced, this looks like the following:

```Go
package data

type Config struct {
	DB      DBConfig     `json:"db"`
	Cache   CacheConfig  `json:"cache"`
	Search  SearchConfig `json:"search"`
}

// DefaultConfig returns the default configuration for the data layer
func DefaultConfig() *Config {
	return &Config{
		DB: DBConfig{
			Address:  "127.0.0.1:28015",
			Database: DatabaseName,
			Timeout:  "60s",
		},
		Cache: CacheConfig{
			Addresses: []string{"127.0.0.1:11211"},
		},
		Search: SearchConfig{
			Addresses:  []string{"http://127.0.0.1:9200"},
			MaxRetries: 0,
			Index: SearchIndexConfig{
				Name:     "townsourced",
				Shards:   5,
				Replicas: 1,
			},
		},
	}
}

func Init(cfg *Config) error {
	// initialize rethinkdb, memcached and elasticsearch clients based on passed in config
}

```

Config is then loaded and passed into the `data` layer:

```Go
// main.go

package main

func main() {
	...

	settingPaths := config.StandardFileLocations("townsourced/settings.json")
	cfg, err := config.LoadOrCreate(settingPaths...)
	if err != nil {
		app.Halt(err.Error())
	}

	...

	dataCfg := data.DefaultConfig()

	err = cfg.ValueToType("data", dataCfg)
	if err != nil {
		app.Halt("Error reading data config values: %s", err.Error())
	}

	err = cfg.Write()
	if err != nil {
		app.Halt("Error writing config file to %s. Error: %s", cfg.FileName(), err)
	}

	err = data.Init(dataCfg)
	if err != nil {
		log.Fatalf("Error initializing townsourced data layer: %s", err.Error())
	}

	...
}

```

## The Web Package

The **web** package is the client's entry point to your application.  Here you will extensively use the [http package](https://golang.org/pkg/net/http/) 
and the [encoding/json package](https://golang.org/pkg/encoding/json") for any REST APIs.

You will process cookies at this level and pass already extracted session identifies to your application layer.  The http
package and its types and interfaces (requests, writers, cookies, etc.) should never leave this package.  Or to put it
another way, you shouldn't need to import http in *any* other packages except this one.  Preventing the http package from 
leaking into other packages will help keep the responsibilities of http handling clearly defined *only* in the web package.

## The App Package

The **app** package is where the meat of your application will be.  Inside this package you'll define exported types
that will be passed to and from your other packages; types like `app.User` and `app.Session`.  The point of the app package
is to validate and handle data within these types, so most of your code in the app package will be methods on types, or
functions that return these types. Be conservative about which types and methods you choose to export.  The web package
doesn't need to understand how an `app.User` is approved to see data, it just needs to know whether they can or not.

More likely than not your application data will be related to each other.  Try to retrieve types from their relationships
with other types rather than from standalone functions.

```Go
	//  instead of this
	func UserFromSession(s *Session) (*User, error) {

	}

	// do this
	func(s *Session) User() (*User, error){

	}

```

If standalone functions are necessary or make more sense (for instance when they don't actually modify any type data), 
then don't export them.

If you aren't careful you may find yourself implementing parts of your application in the web or data layers when they
should be in the app layer; for example, access controls.

You may be tempted to check if a user has access to some data at the web layer, but this type of access control should 
always be done at the app layer, where it can be easily found, and unit tested.  

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

The data package is for getting and setting data in your persistent and temporary storage.  There shouldn't be any types
kept here unless they are types that augment the data layer.  For example, you shouldn't see a `User` or `Session` in the
data layer, but you may see a type called `UUID` that exists within the `app.User` or `app.Session` types.

A specific example of this in [Townsourced](https://www.townsourced.com) is the `data.Version` type which looks like 
this:

```Go
type Version struct {
	VerTag  string   // Unique, random value identifying this version
	Created time.Time // When this record was first created
	Updated time.Time // when this record was last updated
}
```

This Version type is then embedded in other types to easily track when objects are created and updated.

```Go
type User struct {
	Username       data.Key        
	Email          string         
	...
	data.Version
}

```

The real usefulness of `data.Version`, and the reason it exists in the `data` package is its ability to prevent any
updates running in the data layer if the `vertag` being submitted by the user doesn't match the `vertag` in of the
record in the database, preventing your users from updating an entry based on stale data.  

Another example of the data layer's responsibilities is cache invalidation. In Townsourced, the code to determine when to 
invalidate and update *memcached* entries exists in the `data` layer, not the `app` layer.  The `app` layer shouldn't 
worry about *how* data is stored or retrieved, it should only concern itself with the structure and behavior of its types. 

## Putting it all together

At this point your web application should look like this:

```
webapp
|--main.go
|
|--app
|   |--user.go
|   |--session.go
|   |--whatever.go
|   
|--data
|   |--data.go
|   |--version.go
|   
|--web
    |--server.go
    |--json.go

```

* `main.go` imports `web`, `app`, and `data`
* `web` imports `app` and `data`
* `app` imports `data`

# Error Handling

Handling errors properly is important in any application, but especially so in web applications where your users may be
anyone from a 90 year old grandmother trying to get to facebook from your website, or a sophisticated hacker who's bored
and looking for an easy target. 

In web applications, errors tend to happen in one of two ways: the system broke or the user broke the system. Conveniently 
http has already given us status code ranges to differentiate between the two types of errors:

* Status 400 range - User Errors
* Status 500 range - Server Errors

If the user breaks the system by putting in bad input using the web application in a way that wasn't intended, they should
be notified of the mistake with a clear explanation of what went wrong and how.  I refer to these errors as **Failures**.

If the system breaks (due to network, hardware, or system design issues) the user should *not* be given any details. 
Any information returned as to the cause of the error is a potential security issue where you may be leaking private
details of your server architecture, configuration vulnerabilities, or specific library versions which could then be
used to attack your application in a specific way.

To help manage which errors should be visible to users and which shouldn't, I usually create a `fail` package, which
resides outside the existing packages. It exists outside the main packages, because failures may happen in any
of them.  Inside the `fail` package is the `fail.Fail` type which implements the `error` interface, so it can be 
returned like a normal error.

```Go
type Fail struct {
	Message    string      `json:"message,omitempty"`
	Data       interface{} `json:"data,omitempty"`
	HTTPStatus int         `json:"-"` //gets set in the error response
}

func (f *Fail) Error() string {
	return f.Message
}

// IsFail tests whether the passed in error is a failure
func IsFail(err error) bool {
	if err == nil {
		return false
	}
	switch err.(type) {
	case *Fail:
		return true
	default:
		return false
	}
}

```

When errors bubble up to the `web` package, you can use a [type switch](https://golang.org/doc/effective_go.html#type_switch)
to determine which are failures that can be shown to the user.

```Go
errMsg := ""

switch err.(type) {
case *fail.Fail:
	errMsg = err.Error()
case *http.ProtocolError, *json.SyntaxError, *json.UnmarshalTypeError:
	//Hardcoded error types which can bubble up to the end users
	// without exposing internal server information, make them behave like failures
	err = fail.NewFromErr(err)
	errMsg = fmt.Sprintf("We had trouble parsing your input, please check your input and try again: %s", err)
default:
	errMsg = "An internal server error has occurred"
}

```

# Tips and Tricks

## JSON Input

When handling JSON input, such as for an API, be wary of Go's [zero values](https://golang.org/ref/spec#The_zero_value).
In an API you need to be able to tell the difference between setting a value to a zero value, and not setting a value at 
all. Consider the following scenario.  Let's say you have an API where the user can set their Name or the Email Address,
you may define the input like this:

```Go
type UserInput struct {
	Email string
	Name  string
}
```

If a user just wants to update their Name and not their Email, they'll send some JSON like this:

```Javascript
{ Name: "New Name" }
```

Which, when parsed, will result in `Name == "New Name"` and `Email == ""`.  You don't know if the user meant to set their
email to `""` or if they simply didn't specify any change to their Email.

You should instead define your input with pointers:

```Go
type UserInput struct {
	Email *string
	Name  *string
}
```

This way you can check if Email was set to `""` or if it wasn't specified at all:
```Go
if input.Email != nil {
	//update email
}
```

Be careful to check each input though, so you don't run into any nil pointer panics.

## Development

When developing your web application, you need to see immediate feedback from the changes you're making.  This means seeing
the results of your work without having to restart your web server.  For this reason it's important to create a *dev mode* 
in your application that can be turned on via a command line flag, or environment variable.  Once set, you can use this 
variable to rebuild templates on every request, reload files on every request, watch for file changes and auto-reload 
the web page. You'd be surprised how much your productivity can suffer with a long write and evaluate loop.  In my 
opinion it's worth spending time getting this working as soon as possible.

```Go
func (t templateHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if devMode || t.template == nil {
		t.loadTemplates()
	}
}
```

## sync.Pool

When each connection comes in on its own separate go-routine, the easiest way to write your code is create and destroy
everything you need within that go-routine.  This will work for a while, but eventually, if your request load starts 
increasing you'll start running into issues with garbage collection.  Most of the time, the answer to that problem is 
`sync.Pool`.

For example, if you want to gzip every response, your first pass may have you creating a new gzip writer on every request.
If your server gets a huge spike in traffic, then your server will be spending a lot of time reclaiming unused gzip writers
when things start to die down.  

Instead, you can use `sync.Pool` to instantiate your gzip writers, and when you're done with them, put them back in the 
pool to be reused, instead of garbage collected.

```Go
func init() {
	zipPool = sync.Pool{
		New: func() interface{} {
			return gzip.NewWriter(nil)
		},
	}
}

...

// custom response writer
type gzipResponse struct {
	zip *gzip.Writer
	http.ResponseWriter
}

func (g *gzipResponse) Write(b []byte) (int, error) {
	if g.zip == nil {
		return g.ResponseWriter.Write(b)
	}
	return g.zip.Write(b)
}

// http server calls Close() on the responseWriter
// return the gzip writer to the pool
func (g *gzipResponse) Close() error {
	if g.zip == nil {
		return nil
	}
	err := g.zip.Close()
	if err != nil {
		return err
	}
	zipPool.Put(g.zip)
	return nil
}

func responseWriter(w http.ResponseWriter, r *http.Request) *gzipResponse {
	var writer *gzip.Writer
	if strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
		w.Header().Set("Content-Encoding", "gzip")
		gz := zipPool.Get().(*gzip.Writer)
		gz.Reset(w)
		writer = gz
	}
	return &gzipResponse{zip: writer, ResponseWriter: w}
}
```

# Conclusion

As you can see from the length of this post, there are a lot of things to consider when building a web application, and
I've only just scratched the surface of the topic.  Hopefully this post can act as a foundation to start from, as well 
as a base to refer to in future posts about more specific aspects of web application development in Go.

I hope you find this post useful, and if you have any corrections, questions, or suggestions for future posts, let me 
know in the comments below.

\* edit [Part 2: Authentication](/post/anatomy-of-a-go-web-app-authentication) has since been written.


