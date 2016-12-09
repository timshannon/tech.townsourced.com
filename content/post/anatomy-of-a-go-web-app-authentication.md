+++
title = "Anatomy of a Go Web App - Part 2: Authentication"
draft = true
date = "2016-12-08T19:05:51-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference"]
+++

This is part 2 of set of posts breaking down some of the decisions I made when putting together the web server for
[townsourced](https://www.townsourced.com).  The first part is [here](/post/anatomy-of-a-go-web-app/).

Instead of a general overview, like part 1, this post will focus specifically on **User Authentication**, i.e. how to handle
passwords (if at all) and session management.

<!--more-->

# Password Management

The best, and most secure system for managing passwords in a web app is *not to manage passwords at all*.  The most secure
and bulletproof password system is still vulnerable to your user's fallibility, and as you'll see when we go over password
management below, most of the work goes into trying to protect the users from themselves.

So, how do you get out of the password management business?  You offload the work to trusted 3rd parties.  This usually
means integrating [OAUTH](https://en.wikipedia.org/wiki/OAuth) into your authentication code.  You'll want to use popular
3rd parties through which your users will probably already have accounts setup. My policy is to give users as many options
as possible so that the simplest option for them is to *not* use a password.  This means adding OAUTH support for 
**The Big 3**.

* [Facebook](#facebook-oauth)
* [Google](#google-oauth)
* [Twitter](#twitter-oauth)

This will not be a detailed overview of implementing OAUTH.  There are 
[better articles](https://aaronparecki.com/2012/07/29/2/oauth2-simplified) out there for that.  I will, however, step 
you through the pieces you need to have in place in order to get it working.  A very high level overview of OAUTH looks
like this:

1. You direct the `user` to a 3rd Party's OAUTH URL passing them the following:
	* A URL on your server to continue the OAUTH process, referred to from now on as the `OAUTH URL`.
	* A random and unique token you've generated and stored on your server
2. The `user` accepts (or denies) access.
3. The 3rd party redirects your `user` to your `OAUTH URL`.

For each of these, you'll need to store a temporary token on your server.  

```Go
// ThirdPartyState is used both
// for getting the user back to where they were when they logged in, as well as
// CSRF protection
type ThirdPartyState struct {
	Token     string // OAUTH Token
	ReturnURL string // Where the user signed in at, so we can send them back there once authed
	Provider  string // Facebook | Twitter | Google
}
```


## Facebook OAUTH

Facebook uses OAUTH 2.0

## Google OAUTH
## Twitter OAUTH

---

* Authentication
  * No Password: Twitter, Facebook, Google
  * Password
    * sha512 + bcrypt
    * Ideal minimum length 12, more reasonable min length 8, no max (sha512), length over complexity
    * Check matches against top common passwords list (https://github.com/danielmiessler/SecLists/tree/master/Passwords)
  * Accept Username or Email logins
* Session Management
	* Cookies
	* Remember Me
	* Forgot Password
	* Logging out
	* Expiration
* No password hints or security questions
* New NIST guidelines are a good starting point: https://nakedsecurity.sophos.com/2016/08/18/nists-new-password-rules-what-you-need-to-know/

