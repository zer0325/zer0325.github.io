---
layout: post
title: Lab - SQL login vulnerability allowing login bypass
---

### Description:

This lab contains an SQL injection vulnerability in the login function.

To solve the lab, perform an SQL injection attack that logs in to the
application as the administrator user. 

### Solution:

In this lab you will be subverting the application logic of the login function.
The statement that allows login is the following:

`SELECT [columns] from [table] where username = [username] and password =
[password]`

In this lab, there is a user `administrator` which we want to login without
knowing the password. To login the user `administrator` type the following in
the [username] and [password] boxes.

username: `administrator' --`
password: (any password)

The above input has the following sql statement:

`SELECT [columns] from [table] where username = 'administrator' -- and password
= 'test'`

This works because the two dashes (--) at the end of the text `administrator`
effectively comment out the succeeding texts.

![apprentice lab02.png](/assets/images/sql/apprentice_lab02.png)
