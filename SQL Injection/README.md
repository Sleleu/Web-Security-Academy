# SQL Injection

SQL injection (SQLi) allow an attacker to **interfere with the queries that an application makes to its database.**
There are a wide variety of SQLi :

- **Retrieving hidden data :** using an SQLi to return additional results
- **Subverting app logic :** change a query to interfere with the app logic
- **UNION attacks :** retrieving data from other database tables
- **Examining the database :** extract informations about version and structure of the databases
- **Blind SQL injection :** where the results of a query you control are not returned in the app responses

## Retrieving hidden data

The SQL query for : https//website.com:products?category=Gifts 
is : SELECT * FROM products WHERE category = 'Gifts' AND released = 1

= SELECT *(**all** details) FROM products(the products **table**) WHERE category = 'Gifts'(category of the table) AND released = 1(to **hide unreleased products**)

For an insecure website, an attacker can construct an attack to include inreleased products :
https//website.com:products?category=Gifts'--
= SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1

The double dash sequence is a commment indicator that mean the rest of the query is interpreted as a comment.
