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
  - badger
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
simple enough, but as the *individual unanmously voted in charge of all things tech in the house*, I never seem to find
the time to actually complete the task.  This problem gets even worse when you start talking about the digital photo 
frames at the grandparents' house.  Those frames are still cycling the same set of pictures loaded when they were
first gifted.

# The Plan

If I couldn't be bothered to keep the digital photo frame up to date, then instead I was going to build a frame that 
goes out and collects the most recent pictures from the locations that they already exist in.  For my family, this 
meant the following:

* Instagram
* Facebook
* Google Photos
* Email

