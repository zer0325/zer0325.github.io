---
layout: post
title: Lab - SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
---


### Description:

This lab contains an SQL injection vulnerability in the product category filter.
When the user selects a category, the application carries out an SQL query like
the following:

SELECT * FROM products WHERE category = 'Gifts' AND released = 1

To solve the lab, perform an SQL injection attack that causes the application
to display details of all products in any category, both released and
unreleased. 

### Solution

Select a category from the  page. When the page reloads the url will have
`xxx/?Category=Accessories`, assuming that you selected the `Accessories`
category. To solve the lab, add the following in url: `?Category=Accessories' or '1'
= '1' -- `. This works because of the nature of the **SELECT** keyword. **Select**
retrieves the rows in a table or one more tables. The WHERE clause adds a
condition whether it will retrieve the current row. If the condition after the
WHERE keyword evaluates  to true, then the current row will be retrieved. 

The solution above has the following statement:

`SELECT [columns] FROM [table] WHERE category = 'Accessories' or '1' = '1' -- AND
released = 1`

The added condition `'1' = '1'` always evaluates to true, so when ORed will
always evaluate to true whatever the evaluation of other conditions are. The
two dashes (--) at the end means to treat the succeeding text as a comment.

![apprentice lab01](/assets/images/sql/apprentice_lab01.png)

