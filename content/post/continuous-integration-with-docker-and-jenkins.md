---
title: "Continuous Integration with Docker and Jenkins"
draft: true
date: "2019-01-01 18:00:00"
categories: 
  - Development
tags: 
  - Development
  - Continuous Integration
  - Docker
  - Testing
keywords:
  - go
  - golang
  - development
  - best practices
  - gotchas
  - programming
  - continuous integration
  - ci
  - ci/cd
  - jenkins
  - testing
  - docker
  - containers
  - docker-compose 
---


# Notes
* Built in Docker support with Jenkins and it's limitations
* Docker only supports a single process, which makes testing things with multiple processes difficult
* Jenkins pipelines don't support docker compose
* you can use docker compose directly, but make sure it returns the proper error codes
* Struggled with permissions on files created in the docker container not being able to be cleaned up by Jenkins
* Also has issues where concurrent tests would delete files needed by other running tests
  * Resolved both by copy working dir into container-only dir and running against the copy.
* Timezones in Docker - Had a falsely passing unit test due to the fact that Docker's timezone was UTC. 
  * Some database backends were storing UTC some weren't, but once any of the tests ran in Docker, the *current* timezone was always UTC so all tests always passed.
