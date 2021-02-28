---
layout: post
title: Lab - SQL injection UNION attack, finding a column containing text 
---

### Description

 This lab contains an SQL injection vulnerability in the product category
 filter. The results from the query are returned in the application's response,
 so you can use a UNION attack to retrieve data from other tables. To construct
 such an attack, you first need to determine the number of columns returned by
 the query. You can do this using a technique you learned in a previous lab.
 The next step is to identify a column that is compatible with string data.

 The lab will provide a random value that you need to make appear within the
 query results. To solve the lab, perform an SQL injection UNION attack that
 returns an additional row containing the value provided. This technique helps
 you determine which columns are compatible with string data. 

### Solution

 As in the previous lab, find first the number of columns returned by the
 original query using the `union select null` statement. It turns out that the
 number of columns is three. Note that if the number of columns is not three as
 determined by the `union select null` statement, it will return an `internal
 server error`. Once the number of columns  is determined, change the payload
 to test for the column that has a string type.

 `' UNION SELECT 'a', null, null --`

 The lab provides a `text` to test to solve the lab. In this case it is
 `RuWM4x`.

 ![practitioner lab 02 text test](/assets/images/sql/practitioner_lab02_test_text.png)

It turns out that the second column has the string type. The following
statement solves the lab:

`xxxxx/?category=' UNION SELECT null, 'RuWM4x', null --`

![practitioner lab 02 result](/assets/images/sql/practitioner_lab02.png)

