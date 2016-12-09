+++
title = "Anatomy of a Go Web App - Part 2: Authentication"
draft = true
date = "2016-12-08T19:05:51-06:00"
categories = ["Development"]
tags = ["Go", "Web Development", "tutorial", "reference"]
+++

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

