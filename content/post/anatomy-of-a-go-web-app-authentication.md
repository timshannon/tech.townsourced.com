+++
title = "Anatomy of a Go Web App - Part 2: Authentication"
draft = true
date = "2016-12-08T19:05:51-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference"]
keywords = ["web development", "go", "golang", "backend development", "passwords", "oauth", "security", "best practice"]
+++

This is part two of set of posts breaking down some of the decisions I made when putting together the web server for
[townsourced](https://www.townsourced.com).  The first part is [here](/post/anatomy-of-a-go-web-app/).

Instead of a general overview, like part one, this post will focus specifically on **User Authentication**, i.e. how to handle
passwords (if at all) and session management.

<!--more-->

# Password Management

The best, and most secure system for managing passwords in a web app is *not to manage passwords at all*.  The most secure
and bulletproof password system is still vulnerable to your user's fallibility, and as you'll see when we go over password
management below, most of the work goes into trying to protect the users from themselves.

## OAUTH

So, how do you get out of the password management business?  You offload the work to trusted 3rd parties.  This usually
means integrating [OAUTH](https://en.wikipedia.org/wiki/OAuth) into your authentication code.  You'll want to use popular
3rd parties through which your users will probably already have accounts setup. My policy is to give users as many options
as possible so that the simplest option for them is to *not* create a password in my app.  This means adding OAUTH 
support for one or more of the **Big Three**.

* **Facebook**: [Manually Build a Login Flow](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow)
* **Google**: [Google Sign-In for server-side apps](https://developers.google.com/identity/sign-in/web/server-side-flow)
* **Twitter**: [Implementing Sign in with Twitter](https://dev.twitter.com/web/sign-in/implementing)

This will not be a detailed overview of implementing OAUTH, there are 
[better articles](https://aaronparecki.com/2012/07/29/2/oauth2-simplified) out there for that.  I will, however, step 
you through the pieces you need to have in place in order to get it working.  A very high level overview of OAUTH looks
like this (courtesy of Google):

![OAUTH2 Server Side Flow](/images/post/anatomy-of-a-go-web-app-authentication/server_side_code_flow.png "Server Side Code Flow")

Facebook and Google are very similar, and you should be able to get by with a standard 
[OAUTH2 library](https://github.com/golang/oauth2) (with a little bit of tweaking).  Twitter, on the other hand, actually 
uses *OAUTH1.0A*, so you'll need to make sure you use a library built for it like 
[https://github.com/mrjones/oauth](https://github.com/mrjones/oauth).

Each of the **Big Three** have a unique user identifier that you can store with your [user record](/post/anatomy-of-a-go-web-app/#the-app-package).

```Go
type User struct {
	Username       data.Key        
	Email          string          
	...
	GoogleID       string         
	FacebookID     string        
	TwitterID      string       
	...
}
```

And once that's in place, you can do something like this:
```Go
// FacebookUser gets a user (if possible) from the passed in facebook code
func FacebookUser(redirectURI, code string) (*User, error) {
	fbSes, err := facebookGetSession(redirectURI, code)
	if err != nil {
		return nil, err
	}
	//Lookup user in the database
	usr, err := userGetFacebook(fbSes.userID)
	if err == ErrUserNotFound {
		data, err := fbSes.userData()
		if err != nil {
			return nil, err
		}

		// Create new user from Facebook data
		...
		return newUser, ErrUserNotFound
	}
	if err != nil {
		return nil, err
	}

	// Return user and log them in
	return usr, nil
} 
```

Your Google and Twitter functions will look very similar.  Check if a user already exists, associated to the given 
third party credentials, and if so, log them in, otherwise use the information shared (or accessible via some other API)
to build a new user.


## Passwords

If for some reason your user does not want to use a 3rd party login, or you aren't permitted to offer them in your app
(for instance internal apps), you'll need to securely manage your user's passwords.

### Requirements

There is a lot of discussion, and argument over password best practices, and I highly recommend doing your own research,
and getting a good understanding of the implications of some of these decisions.  Once again, the best option is to 
*not to manage passwords at all*. That being said, here is a short list of best practices I try to meet when setting 
my password requirements.

1. Longer passwords are better than complex passwords
	* Skip doing any complexity checks, or requiring a minimum level of entropy, or at least one number, 
	or special symbol, etc
	* Minimum password length should be 10, or even better 12 (as of 2016).
	* No max length (more on handling that later)
2. Password should not be on the top 1,000 (or more) most common passwords list
	* Load a text file during the `Init()` of your [app layer](/post/anatomy-of-a-go-web-app/#the-app-package) and
	test new passwords against it.
	* Update the password file yearly. A good source I've found is [here](https://github.com/danielmiessler/SecLists/tree/master/Passwords).

Thats it. 

Simple requirements are easier for your users to understand, easier for you to manage, and easier for you to
rip out and rewrite when technology inevitably changes.

Recommended reading:

* [New NIST Guidelines for Passwords](https://nakedsecurity.sophos.com/2016/08/18/nists-new-password-rules-what-you-need-to-know/) 
* [Your Password is Too Damn Short](https://blog.codinghorror.com/your-password-is-too-damn-short/)

### Implementation




---

* Authentication
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

