+++
title = "Townsourced Open-sourced"
date = "2017-11-28"
categories = ["Releases"]
tags = ["Go", "Web Development", "Townsourced", "Townsourced"]
keywords = ["web development", "go", "golang", "backend development", "open-source", "community", "Townsourced"]
+++

**TLDR: Grab the code here: https://github.com/timshannon/townsourced and run `docker-compose up`**

My goal with Townsourced was first and foremost to build something useful for local, small communities.  I felt local
communities were very under-represented online, and the social networks available catered either to large communities or niche
communities of shared interest.  My target audience was small, local communities: small towns, college campuses, 
neighborhoods, churches, schools.  I wanted to build something that handled the overlap between those local communities
better than existing options.

<!-- more -->

It started as a small side project, and grew from there.  I showed it to a few angel investors, and demoed it to the
Minneapolis Tech community at [Minnedemo](https://minnestar.org/minnedemo/), but it never went any further than that.

I firmly believe that open-source is important, and I want to contribute back in any way I can. There are a few 
interesting things in Townsourced's code that I hope will, at a minimum, be useful as an example.

You can access a live version of this code running at [https://www.townsourced.com](https://www.townsourced.com) which 
I plan to keep running.

If you have any questions, or would like some help setting Townsourced up to run for your own community, feel free to
send me a message at [tim@townsourced.com](mailto:tim@townsourced.com).

# Running Townsourced

The quickest way to run Townsourced is with Docker.  Install [Docker](https://www.docker.com/) and 
[Docker-Compose](https://docs.docker.com/compose/install/), then run `docker-compose up`.

This will start a containerized instance of Townsourced listening on port `8080`, running with RethinkDB, Elasticsearch
and Memcached.

The Docker Compose file can be found [here](https://github.com/timshannon/townsourced/blob/open-source-post-links/docker-compose.yaml).

# Building

Townsourced is written in Go, and all of it's dependencies are vendored.  You can build Townsourced by simply running
`go get github.com/timshannon/townsourced`.

To build the web / client portion you'll need to install [gobble](https://github.com/gobblejs/gobble)

`npm install -g gobble-cli`

Then run `gobble build static -f` in the `web` folder.

# Deploying

The easiest way to deploy Townsourced is to use the Docker.  The `docker-compose.yaml` file can be used to run the
entire stack, or give you an insight into what services are needed.  Simply run `docker-compose up` and you'll have
a running instance of Townsourced at http://localhost:8080.

To deploy Townsourced manually, you'll need:

* [RethinkDB](https://rethinkdb.com/)
* [Elasticsearch](https://www.elastic.co/products/elasticsearch) (2.4.6)
* [Memcached](https://memcached.org/)

Townsourced will look for a `settings.json` file either in it's running directory, or in `/etc/townsourced/` on linux. In
that `settings.json` file will be the connection parameters that Townsourced will use to connect to the various services.
Here is an example of what that looks like:
```json
{
    "app": {
        "httpClientTimeout": "30s",
        "taskPollTime": "1m",
        "taskQueueSize": 100
    },
    "data": {
        "cache": {
            "addresses": [
                "127.0.0.1:11211"
            ]
        },
        "db": {
            "address": "127.0.0.1:28015",
            "database": "townsourced",
            "timeout": "60s"
        },
        "search": {
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
        "address": "https://www.townsourced.com",
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

Finally, you'll need a `web/static` folder (built from gobble) in the running directory of Townsourced.

# Items of Interest
I've dived into more detail in the [Anatomy of a Go Web Application](/post/anatomy-of-a-go-web-app/) and 
[Authentication](/post/anatomy-of-a-go-web-app-authentication/) posts, but I wanted to quick run through a few points
of interest (in my opinion) in the Townsourced code.

## Posts that exist in more than one town
One of my big goals with Townsourced was to better describe the overlapping communities that a person belongs to. Most
social networks simply have Groups that you join.  Townsourced *Towns* are similar except that when you create a post,
you can post it to multiple communities *simultaneously*.  The discussions around that post live with the post and exist
across communities, but communities could still moderate a post in one Town without it impacting another town.

![Multi-town Posts](/images/post/townsourced-opensourced/multi-town.png "Multi-town Posts")

You can see the Post structure, and how it tracks Towns and moderation 
[here](https://github.com/timshannon/townsourced/blob/open-source-post-links/app/post.go#L99).

## Private package for open source release
I knew I wanted to eventually open-source Townsourced regardless of whatever came of it, so I made sure that I kept
anything that I didn't want to be released to the public (API keys, passwords, etc) in a separate package so that it
could be easily removed.

All of the API keys that Townsourced uses are stored here, from Facebook to the the client side Google Maps API key,
which is [inserted into the html template](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/src/townSearch.template.html#L168).

You can see that package [here](https://github.com/timshannon/townsourced/blob/open-source-post-links/data/private/private.go).

If you want to run Townsourced for yourself you can get each one of the API keys used for free without committing to any
subscription or fees.  The [Google Maps API key](https://developers.google.com/maps/documentation/embed/get-api-key)
would be a minimum requirement to have for a lot of the UI functionality to work.

## Attribution
Open-source is important, and Townsourced relies heavily on open-source technologies.  Most open-source licenses are
incredibly permissive, but they also have a clause denoting the need for attribution to the original author. 

Most people simply include the license or creator comments with their releases, which meets the requirements of the license.
What I try to do in any software I write is find a place to clearly list those attributions so they are easy to find. It
also makes it easy to see the technologies used to build the piece of software.
In Townsourced, a link to [that attribution](https://www.townsourced.com/attribution/) can be found in the footer of every page.

## Versioning
In most web applications you need to be able to handle the scenario where two users are updating the same data.  One
will finish their changes and save, and the second user will then be editing the *previous* version of that data.  If you
allow the second user to commit their changes, the first user's data will be overwritten.  Sometimes this isn't a big deal,
but I chose to not allow it in Townsourced.

I handled this by generating a small random value with any object I wanted to version, and when a database update happens
it checks if the current version matches the passed in version with the update.  If the version doesn't match, the user gets
a message saying their screen is out of date and they need to refresh the page.

Go makes it easy to embed this [version struct](https://github.com/timshannon/townsourced/blob/open-source-post-links/data/version.go#L42)
that handles the update logic into [existing structs](https://github.com/timshannon/townsourced/blob/open-source-post-links/app/town.go#L22).

You can see how that works [here](https://github.com/timshannon/townsourced/blob/open-source-post-links/data/version.go).

## Less in Ractive Components
When I first started Townsourced, [Vue](https://vuejs.org/) was just getting rolling, and I hadn't even heard of it yet.
I had built a few projects with [Ractive](https://ractive.js.org/) and found it to be simple and powerful.  I especially
loved Ractive Components; bundles of application UI logic combined with encapsulated HTML and CSS.  It made it really easy
to build a UI in pieces and reuse them as needed.

However, I quickly ran into issues where I was having to duplicate many of the variables that were defined in my `.less` files
into my components.  Luckily my build tool of choice ([Gobble](https://github.com/gobblejs/gobble)) made it easy to write
a small section into my build scripts to let me compile less into my Ractive components.  This is, of course, something 
that the Vue ecosystem now handles automatically for you.  This flexibility is also a reason why I will always favor a 
programmable build tool over a configuration based one.

You can see that [here](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/gobblefile.js#L138) and how it was used
[here](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/src/js/components/post.html#L74).

-----------------------

There are many other things I could cover with more detail, like:

* [Automatically generating tiny, blurred placeholder images](https://github.com/timshannon/townsourced/blob/open-source-post-links/data/image.go)
to [show when posts are loading](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/src/js/components/image.html)
* Easy [sharing of existing posts](https://github.com/timshannon/townsourced/blob/open-source-post-links/app/shareParsers.go)
 from other sites [with Townsourced](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/src/js/bookmarklet.js)
* Making a large [interactive SVG for the Pitch page](https://github.com/timshannon/townsourced/blob/open-source-post-links/web/src/pitch.html)
* How to do [nested comments with Rethinkdb](https://github.com/timshannon/townsourced/blob/open-source-post-links/data/comment.go)

But I'll save those details for possible future posts.  If anyone is interested, [let me know](mailto:tim@townsourced.com).

Hopefully there are a few things in the repository that you may find useful.  If so, I'd love to hear about it.  Either
way it's out there now, and under an [MIT license](https://github.com/timshannon/townsourced/blob/open-source-post-links/LICENSE), for 
you to do with it as you like.
