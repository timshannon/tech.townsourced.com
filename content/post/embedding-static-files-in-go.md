---
title: "Choosing A Library to Embed Static Assets in Go"
draft: false
date: "Thu Apr 26 20:58:55 CDT 2018"
categories: 
  - Development
tags: 
  - Go
  - Web Development
  - reference
  - Continuous Integration
  - Lex Library
keywords:
  - web development
  - go
  - golang
  - development
  - passwords
  - CI
  - Continuous Integration
  - Lex Library
  - best practices
  - gotchas
---

# Background
One of the oft-touted benefits of Go is that applications written in it are easily deployed because they are 
statically complied.  A lot of this benefit goes away if you need to manage the location and permissions on a bunch of
files needed to run a web application.

The solution is to compile any necessary files into the application binary itself.  This can be done in Go by using
a byte slice literal containing the [string representation](https://golang.org/ref/spec#String_literals) of the bytes in
a file.

```Go
 fileData := []byte("\x1f\x8b\ ... \x0f\x00\x00")
```

The biggest downsides to this approach are:

* **Larger binaries**
	* For my current project [Lex Library](https://github.com/lexlibrary/lexlibrary) I'm seeing a 20 MB executable without embedded 
	assets, and a 21 MB executable with them.
* **Longer compilation**
	* This is mostly mitigated by the [latest caching](https://golang.org/doc/go1.10#build) in the Go compiler.
* **Increased memory during compilation**
	* This may bite you if you are developing on low memory devices, but for me personally this hasn't been a concern.

You are essentially making a trade-off between development time and operational management time.  If your application's intended
audience is the general public (or at least the really geeky public that host their own web apps), this trade off is more
than worthwhile.

# Embedding Options
Perhaps the first library, or at least the first really popular one, to handle embedding static assets in Go applications
is [jteeuwen's Go-BinData](https://github.com/jteeuwen/go-bindata).  It's a command line application that you pass a
directory path to and it generates a `.go` file with your assets embedded in it.  

Unfortunately, jteeuwen seems to have dropped off the face of the planet, deleting all of this repositories from github 
as he left.  Luckily for all of us, his code was open-source and so was forked by the community at large. You can find 
several, well-maintained forks of his code on github.  My fork of choice was [shuLhan's](https://github.com/shuLhan/go-bindata), 
but I have since moved onto other options, the reasons for which, I'll get into below.

More details on the jteewen repo are here: https://github.com/jteeuwen/go-bindata/issues/5 

## Alternatives
Since jteewen's library there have been many, many alternatives.  Below is a *non-comprensive* list I put together 
while researching this myself:

* **vfsgen** - https://github.com/shurcooL/vfsgen
* **go.rice** - https://github.com/GeertJohan/go.rice
* **statik** - https://github.com/rakyll/statik
* **esc** - https://github.com/mjibson/esc
* **go-embed** - https://github.com/pyros2097/go-embed
* **go-resources** - https://github.com/omeid/go-resources
* **packr** - https://github.com/gobuffalo/packr
* **statics** - https://github.com/go-playground/statics
* **templify** - https://github.com/wlbr/templify
* **gnoso/go-bindata** - https://github.com/gnoso/go-bindata
* **shuLhan/go-bindata** - https://github.com/shuLhan/go-bindata
* **fileb0x** - https://github.com/UnnoTed/fileb0x
* **gobundle** - https://github.com/alecthomas/gobundle
* **parcello** - https://github.com/phogolabs/parcello

The main goal of this post is to help you sort through differences and help you determine which features you should consider
when choosing one of these libraries.

# Separating the Wheat from the Chaff
With so many options, it can be overwhelming to determine which library will work best for your purposes. While your
application may have different requirements than mine, chances are if it's a web application, there will be a lot
of overlap.  Hopefully this comparison of the libraries below will be useful if you need to make this
same decision.

## Criteria

### Compression
As mentioned before, one of the downsides to embedding static files is that it increases the size of the executable.
You can alleviate some of that by using a library that compresses the files before embedding them.  It is usually worth
the small cost of decompression to save space as well as less memory usage when compiling.  Also, static web files tend to
compress really well (other than images), which is why most web browsers support receiving
gzip compressed files.  This leads directly to my next criterion.

### Optional Decompression
If your static files are stored in your executable with gzip compression, and you are going to potentially serve up those
files to your client with gzip compression, why not just send them the already compressed file data?  The ideal library
would give you the option, at runtime, to retrieve the already compressed file without decompressing it first.

### Loading from the local File System
When you are developing your web application, anything that adds time or friction between when you *make* a change and
when you *see* a change in your app, should be limited.  If you have to rebuild your statically embedded go file every
time you make a change to CSS or HTML, you'll quickly be looking for alternatives.  

The ideal library should allow you to easily switch between a development build, where the files are loaded locally and 
on-the-fly, and your production build, where the files are fully embedded into executable and ready for distribution.

### Reproducible Builds
This criterion took me by surprise, and I didn't consider it originally when I started development on 
[Lex Library](https://github.com/lexlibrary/lexlibrary). As mentioned earlier, my first choice was a [go-bindata fork by
shuLhan](https://github.com/shuLhan/go-bindata).  I chose it mainly because I was already familiar with the original
go-bindata library, and this fork looked well maintained.  

The library was working great, but, out of the blue, my CI builds started failing.  Like every developer when their tests
start failing, my first thought was ***what changed***.  I immediately checked my last commit, and tried to figure out how that
change broke my template handling.  After scratching my head for a while, unable to find a culprit, I re-ran my test 
suite against the last successful build and found that those *too* were failing.  That told me the change
had to be environmental, and *not* in my code.  However, that raised more questions.  

I run my Continuous Integration tests in [Docker containers](https://www.docker.com/).  The environments should be 
self-contained, pristine, and reproducible. Somewhere my assumptions about that were wrong.  Stepping through my 
`Dockerfile`, I found my culprit:

```docker
RUN go get -u github.com/shuLhan/go-bindata/...
```

There was a small update to the go-bindata library that broke how I was passing in the path to my static files.  All
of a sudden my embedded files weren't on the paths I was expecting.  Now, this could be blamed on several factors such
as the fact that go get always gets the default branch.  In the end, it came down to the simple truth that a process which was
generating code in my application wasn't being versioned and tracked inside my git repo.  External changes on a dependency
of mine could break my builds or my tests without me knowing.

One way to fix this issue was to store a copy of the pre-compiled go-bindata executable in my git repo, but:

1. It's usually not a good idea to store binary blobs in git.
2. I'd have to manually update it every time there was a bug fix.
3. It would make it much harder to build on any other platform other than the one I develop on.

Alternately, I could find a library that *didn't* require a stand-alone executable, and relied entirely on code that
is vendored in my git repo. This meant a library that supported `go generate`, and more specifically, one that didn't 
rely on an external executable to run.

### Additional Criteria
As I mentioned before, you may have different requirements than I do, so in my comparison table below I included a
few additional criteria that you may find useful.

#### Config File
If you have a lot of different folders and files to manage, it can be easier to manage them via a config file checked 
into your source control.

#### http.FileSystem Interface
Using a library that satisfies the [http.FileSystem](https://golang.org/pkg/net/http/#FileSystem) interface can make it
much easier to work with the embedded files.

#### Greater than 200 Github Stars
While this is a bit arbitrary, and it's not *necessarily* a measure of quality, a repo's number of stars *can* be a good
indicator of an active library, or at least one that's used in a lot of	places. This, in turn, *could* mean that a lot of
people are battle testing it, and / or submitting bug reports. The library I chose, just *barely* made it past this 
arbitrary number of stars, so keep that in mind.


# Comparison 

| Library | [Compression](#compression)	| [Opt. decompression](#optional-decompression)	| [Local FS](#loading-from-the-local-file-system) | [go generate](#reproducible-builds) | [No EXE](#reproducible-builds) | [Config File](#config-file)	| [http.FS](#http-filesystem-interface)	| [> 200 Stars](#greater-than-200-github-stars)	|
| -----------------------------------------------------		| ------------- | -------------	| ------------- | ------------- | -------------	| -------------	| ------------- | -------------	|
| **[vfsgen](https://github.com/shurcooL/vfsgen)**  		| **YES**	| **YES** 	| **YES**\*	| **YES**	| **YES**	| NO		| **YES**	| **YES**	|
| **[go.rice](https://github.com/GeertJohan/go.rice)** 		| **YES**	| NO	 	| **YES**	| NO		| NO		| NO		| **YES**	| **YES**	|
| **[statik](https://github.com/rakyll/statik)** 		| **YES**	| NO	 	| NO		| NO		| NO		| NO		| **YES**	| **YES**	|
| **[esc](https://github.com/mjibson/esc)**	 		| **YES**	| NO	 	| **YES**	| **YES**	| NO		| NO		| **YES**	| **YES**	|
| **[go-embed](https://github.com/pyros2097/go-embed)**		| **YES**	| NO	 	| **YES**\*	| NO		| NO		| NO		| NO		| NO		|
| **[go-resources](https://github.com/omeid/go-resources)**	| NO		| NO	 	| **YES**\*	| NO		| NO		| NO		| **YES**	| NO		|
| **[packr](https://github.com/gobuffalo/packr)**		| **YES**	| NO	 	| **YES**	| NO		| NO		| NO		| **YES**	| **YES**	|
| **[statics](https://github.com/go-playground/statics)**	| **YES**	| NO	 	| **YES**	| **YES**	| NO		| NO		| **YES**	| NO		|
| **[templify](https://github.com/wlbr/templify)**		| NO		| NO	 	| **YES**	| **YES**	| NO		| NO		| NO		| NO		|
| **[gnoso/go-bindata](https://github.com/gnoso/go-bindata)**	| **YES**	| NO	 	| NO		| NO		| NO		| NO		| NO		| **YES**	|
| **[shuLhan/go-bindata](https://github.com/shuLhan/go-bindata)**| **YES**	| NO	 	| **YES**	| NO		| NO		| NO		| NO		| NO		|
| **[fileb0x](https://github.com/UnnoTed/fileb0x)**		| **YES**	| **YES** 	| **YES**	| **YES**	| NO		| **YES**	| **YES**	| **YES**	|
| **[gobundle](https://github.com/alecthomas/gobundle)**	| **YES**	| NO	 	| NO		| NO		| NO		| NO		| NO		| NO		|
| **[parcello](https://github.com/phogolabs/parcello)**		| **YES**	| NO 	 	| YES 		| **YES**	| NO 		| NO  		| **YES**	| **YES**	|

\* *Additional code required*

My experience with these libraries vary from writing and deploying applications with them, to simply looking through
their `README` and documentation.  If you find any inaccuracies in the table, please let me know in the comments and 
I'll update them.

# My Choice

The comparison table makes it pretty clear why I ended up going with [vfsgen](https://github.com/shurcooL/vfsgen), and I 
highly recommend it, especially if you require reproducible builds.  A close runner up was 
[fileb0x](https://github.com/UnnoTed/fileb0x), but unfortunately, it's usage of `go generate` required a stand-alone
executable to run.
