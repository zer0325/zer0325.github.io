---
layout: post
title: Lab - SQL injection UNION attack, determining the number of columns returned by the query
---

### Description

 This lab contains an SQL injection vulnerability in the product category
 filter. The results from the query are returned in the application's response,
 so you can use a UNION attack to retrieve data from other tables. The first
 step of such an attack is to determine the number of columns that are being
 returned by the query. You will then use this technique in subsequent labs to
 construct the full attack.

 To solve the lab, determine the number of columns returned by the query by
 performing an SQL injection UNION attack that returns an additional row
 containing null values. 

### Solution

In the start page, select a category .e.g. Pets. This will take you to a page
with the following url: `xxxxxx/?Category=Pets`.

To determine the number of columns returned by the query we can add the
following UNION statement in the url:

`xxxxxx/?Category=' UNION SELECT null --`

Without the `Pets` as a category will also work since it just means that you
are querying nothing. The above has the following sql statement:

`SELECT [columns] from [tables] where category = ' ' UNION SELECT null --`

This will return an internal server error which means that number of columns in
our UNION SELECT statement does not match the original number of columns being
queried. Continue doing this will eventually lead to:

`SELECT [columns] from [tables] where category = ' ' UNION SELECT null, null, null --`

which means that the original number of columns queried is three.

![practitioner lab 01](/assets/images/sql/practitioner_lab01.png)


## Reference:

[https://portswigger.net/web-security/sql-injection/union-attacks](https://portswigger.net/web-security/sql-injection/union-attacks)
