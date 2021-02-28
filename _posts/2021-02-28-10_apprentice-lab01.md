---
layout: post
title: Lab - Reflected XSS into HTML context with nothing encoded
---


### Description

 This lab contains a simple reflected cross-site scripting vulnerability in the
 search functionality.

 To solve the lab, perform a cross-site scripting attack that calls the alert
 function. 


### Solution

According to the descripton above the search functionality is vulnerable. To
solve the lab put `<script>alert('test')</script>` on the search bar and press
the Search button. An alert box will pop as shown below.

![10-apprentice lab01 xss alert](/assets/images/xss/10_apprentice_lab01_xss_alert.png)

The image above shows that the `script` in the search bar is run which confirms
that the site is vulnerable to reflected xss attack.

![10-apprentice lab01 result](/assets/images/xss/10_apprentice_lab01.png)


