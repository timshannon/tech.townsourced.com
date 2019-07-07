---
title: "Building an Always Up-to-Date Digital Photo Frame"
draft: true
date: "2019-06-09 19:36:00"
categories: 
  - Releases
tags: 
  - Development
  - Go
  - programming
  - project
  - raspberry-pi
keywords:
  - go
  - golang
  - raspberry-pi
  - development
  - programming
  - projects
  - boltdb
  - bolt
  - bolthold
  - database
  - embedded
  - web scraping
  - instagram
  - google photos
---
With the release of the new [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/), I was 
once again reminded that I have an old raspberry pi in my drawer collecting dust; the remnant of past, unfinished
projects.  At this point, building a digital photo frame with a Raspberri Pi is so common, it probably qualifies as
a cliché, but I had yet to find a digital photo frame you could buy (or build) whose pictures didn't quickly get out
of date.

# The Problem

The whole point of a digital photo frame is so that you have current, up-to-date pictures, rather than the static
printed ones you'd see in a normal frame.  However, updating those pictures usually involves dumping photos from your
camera or phone onto the sd card or USB drive storing the photos for the digital photo framer. The process seems
simple enough, but as *the individual unanimously voted in charge of all things tech in the house*, I never seem to find
the time to actually complete the task.  This problem gets even worse when you start talking about the digital photo 
frames at the grandparents' house.  Those frames are still cycling the same set of pictures loaded when they were
first gifted.

# The Solution

If I couldn't be bothered to keep the digital photo frame up to date, then instead I was going to build a frame that 
goes out and collects the most recent pictures from the locations that they already exist in.  For my family, this 
meant the following:

* Instagram
* Facebook
* Google Photos
* Email

Email was pretty straightforward, but the other three would each have their own difficulties to surmount.

# The Plan

The design I settled on would consist of a server that both hosts a webpage that cycles between images, and a process
that polls for new images at regular intervals.  A Raspberry Pi should be able to easily run both the server and a 
browser displaying the images.  I also just happened to know a [quick and simple KV store](https://github.com/timshannon/bolthold) 
I could use to store and manage the images on the server.  The tricky part would be collecting the images.  I needed 
image collecting solutions that required very little to no setup, and were robust enough to keep working with little 
maintenance.  This meant API's that required an access token that may expire in the future were a non-starter.  After 
some quick browsing through the aforementioned source's API's, I quickly ran into some roadblocks.

* **Instagram** doesn't allow "automated services of any kind", and seems to be entirely targeted at assisting advertisers
* **Facebook** has completely shut down any public API access since the 
[Cambridge Analytica](https://en.wikipedia.org/wiki/Cambridge_Analytica) scandal, and now requires pre-approval of
applications before they can access even publically posted images.
* **Google Photo's** API doesn't allow any "service" accounts to access the API, and it can only be access via temporary
browser-based OAUTH popups.

Email, on the other hand, just works.

## Web Scrapping to the Rescue
I wasn't going to have the grandparents click an OAUTH pop up every time they wanted to view an image of their grandkids.
I also didn't want to have to store a hardcoded username and password in a config file on an SD card sitting in a photo
frame on Grandma's kitchen counter.

With Instagram, any public profile can be viewed without logging in.  The same applies to a shared Google Photo album 
where you know the secret URL.  Facebook, on the otherhand, doesn't allow any access (at least that I could find),
without first having a Facebook account.  This meant that I could collect *nearly* all the photo's I wanted by scraping
the data instead of using the nice, organized and documented APIs that where handicapped on purpose.

The downside to web scraping is that it's brittle.  Any major structural changes to the web pages, and the scrapers
would break.  I planned on combating this eventuality in 3 ways:

* Write the scrapers as generic as possible 
* Target elements via aria attributes, as they are used by screen readers, and big companies *usually* try to support them
* Build a process to automatically pull the latest scraper code from the git repository on a regular basis, so that
if something does break, it should start working again shortly after I update the main repository.


