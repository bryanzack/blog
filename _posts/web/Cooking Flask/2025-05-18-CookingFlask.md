---
layout: post
title: "Cooking Flask"
date: 2025-05-18
---

### About

Upon visiting the site, we are met with a website that allows you to search for possible recipes. The question on the challenge website implies that we need to steal the admin's password, and that the challenge website "creator" does not know much about databases or coding. 



### The Challenge

The only functional page of this website is the `/search` endpoint, heavily implying that the intended exploitation path is through this page. 

![asdf](https://i.imgur.com/OTvXH2h.png)



Without knowing any of the "recipes" to search for, the only way we can use the site as intended while still yielding results from the search is to use the Tags checkbox fields to search with. We can check all the boxes and see what comes back. 

![tag_results](https://i.imgur.com/gJcluVM.png)



Now that we know how the site is intended to be used, we can start trying to figure out how to break it. From the challenge description we should be thinking about SQL injection first. 





### Finding Injectable Parameters

As a start I thought I'd try and and use `sqlmap` to test all the possible parameters. To do this we first need to know what all the parameters are that get processed by the SQL statements used by the backend. By putting bogus data in the text fields and checking all of the Tags boxes, we can send this request through Burp Suite to get a better look at how this request looks.

![burp_intercept](https://i.imgur.com/KCgKTbp.png)



Now that we have a better idea of how the request looks, we need to use `sqlmap` to test each parameter individually to see. Since the `tags` parameter is redundant, we will remove all instances of it except for one and then use the resulting URL in our `sqlmap` command:

```bash
sqlmap --level=5 --risk=3 --url="https://cooking.chal.cyberjousting.com/search?recipe_name=asdf&description=asdf&tags=Dessert"
```



After a bit of waiting I end up seeing that, according to `sqlmap`, the only injectable parameter seems to be `tags`. I am also told that the DBMS used on the backend is `SQLite`. So to specifically target the `tags` parameter and the `SQLite` DBMS I change my command to the following:

```bash
sqlmap --level=5 --risk=3 --url="https://cooking.chal.cyberjousting.com/search?recipe_name=asdf&description=asdf&tags=Dessert" -p tags --dbms=SQLite
```



After more waiting it seems that all the injection attempts are met with HTTP code 500 Internal Server Error. This means that the `tags` parameter might be injectable, but `sqlmap` wasn't able to do it. It also means that the backend is breaking in response to the sent payloads. Specifically though, `sqlmap` thought that the parameter was injectable through blind OR boolean payloads. So next we need to manually try to break the backend through URL manipulation with this knowledge.



### Breaking the SQL

Through basic manual testing I was able to break the backend with the payload `' or 1=1` shown in the URL below.

```
https://cooking.chal.cyberjousting.com/search?recipe_name=asdf&description=asdf&tags=Dessert%27%20or%201=1
```



And then the error message it produces:

![boolean_break_sql](https://i.imgur.com/3oVeaFo.png)



This error message produces a ton of valuable information, eventually I stumble upon this payload that properly escapes the SQL and comments out the last bit of it and the query executes just fine without error. `') --`. So now base off of previous training I know that I probably need to make use of a `UNION` select statement along with the correct number of columns to properly dump arbitrary database data.





#### Finding Correct # of Columns

![broke_sql_and_escaped](https://i.imgur.com/3EU3FbA.png)



To figure out the right number of columns to use we will simply modify the following payload and increase the amount of columns to select until the error stops complaining about column counts:

​	`') UNION SELECT 1,2,3,etc --`

![wrong_column_count](https://i.imgur.com/rqnXpbb.png)



Eventually I stumble across the right amount of columns and the error sent back to the frontend changes accordingly:

![right_column_count_wrong_data_type](https://i.imgur.com/uF0UAn0.png)



Now the backend is complaining about our statement having the wrong data type. We need to figure out which field(s) in our payload needs to be a string. Thankfully the error now outlines the field names for the intended statement:

![helpful_error_message](https://i.imgur.com/h4J0kig.png)



Now that we know what fields (columns) the `Recipe` data type is comprised of, we can start to fuzz or guess which fields are required to be a string.



#### Finding Correct Payload

I attempted replacing each integer with its string counterpart (1 turns into '1', 2 into '2', etc.) until I got a different error message. I ended up receiving a different error when I got to changing the 7 to '7':

![different_error_message_yay](https://i.imgur.com/Z1vHY21.png)



Primarily this message tells us that although we know what fields make up the `Recipe` data type, they aren't necessarily in the right order in the URL statement, which is not uncommon in these types of problems. We come to this conclusion by reading the part at the bottom where it says this:

```
Datetimes provided to dates should have zero time - e.g. be exact dates [type=date_from_datetime_inexact, input_value='3', input_type=str]
```



So we know that we should replace our '3' with a `date_from_datetime_inexact` adherent value.

[This](https://docs.pydantic.dev/latest/errors/validation_errors/#date_from_datetime_inexact) link gives us a little more insight as to how it should look, which is something like '2023-01-01'. We will try to replace the '3' with that string and see what the error says now. Which is this:

![more_change_needed_to_payload](https://i.imgur.com/dRd1GKH.png)



Again at the bottom, we see that the field '7' is being interpreted as an `int` but needs to be of type `list`. We modify our payload like shown below and finally receive a promising non-error response:

![right_payload_no_data_sad](https://i.imgur.com/iZ7bgkA.png)



So now that we have all the field data types correct and we can see which select fields are reflected back to us in the resulting HTML, we can start to grab arbitrary data from the database, preferably starting with table and column names.





### Retrieving Arbitrary Data

Each database has its own syntax for how it stores metadata pertaining to its structure, so we need to search online for how we can retrieve this data in a way that's specific to SQLite. Since we only have two fields reflected back to us, we want to start with trying to figure out what databases there are, and what tables each database has so we know where to start looking for credentials.



##### Enumerating Tables

[This](https://stackoverflow.com/questions/5334882/how-to-get-list-of-all-the-tables-in-sqlite-programmatically) post details how we can retrieve information from the `sqlite_master` table.

With the following payload can look at all the tables within the `sqlite_master` database:

`') UNION SELECT '1','2','2023-01-01','4',name,'6','["lol"]',8 FROM sqlite_master where type="table" --`

We will simply replace one of the reflected fields (5 and 6) with the `name` value in an attempt to view all of the table names in the database:

![enumerated_tables_lol!](https://i.imgur.com/S1tzrwL.png)





#### Enumerating Column

Almost there. Now that we have all of the table names, we can replace our `sqlite_master where type="table"` with the name of any of these table names, and then attempt to retrieve column data from that table. We will jump straight to the `user` table as that looks the most promising:

![got usernames!](https://i.imgur.com/b7PjX6O.png)





### Flag

As seen above, simply guessing the column names is my first instinct. We see that we have successfully retrieved the `username` column entries from the `user` table. To get the associated passwords we replace the '6' field with `password`:

![flag](https://i.imgur.com/Z7x2Q83.png)