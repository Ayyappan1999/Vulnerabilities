# SQL INJECTION

### What is SQL injection (SQLi)?

SQL injection (SQLi) is a web security vulnerability that allows an attacker to interfere with the queries that an application makes to its database. It generally allows an attacker to view data that they are not normally able to retrieve. This might include data belonging to other users, or any other data that the application itself is able to access. In many cases, an attacker can modify or delete this data, causing persistent changes to the application's content or behavior.

In some situations, an attacker can escalate a SQL injection attack to compromise the underlying server or other back-end infrastructure, or perform a denial-of-service attack.

### What is the impact of a successful SQL injection attack?
  A successful SQL injection attack can result in unauthorized access to sensitive data, such as passwords, credit card details, or personal user information. Many high-profile data breaches in recent years have been the result of SQL injection attacks, leading to reputational damage and regulatory fines. In some cases, an attacker can obtain a persistent backdoor into an organization's systems, leading to a long-term compromise that can go unnoticed for an extended period.

### SQL injection examples
<pre>  
  There are a wide variety of SQL injection vulnerabilities, attacks, and techniques, which arise in different situations. Some common SQL injection examples include:

  Retrieving hidden data, where you can modify a SQL query to return additional results.
  Subverting application logic, where you can change a query to interfere with the application's logic.
  UNION attacks, where you can retrieve data from different database tables.
  Examining the database, where you can extract information about the version and structure of the database.
  Blind SQL injection, where the results of a query you control are not returned in the application's responses.
</pre>

### Retrieving hidden data
  Consider a shopping application that displays products in different categories. When the user clicks on the Gifts category, their browser requests the URL:

  ### https://insecure-website.com/products?category=Gifts

  This causes the application to make a SQL query to retrieve details of the relevant products from the database:

  ### SELECT * FROM products WHERE category = 'Gifts' AND released = 1
 
<pre>
This SQL query asks the database to return:

all details (*)
from the products table
where the category is Gifts
and released is 1.
The restriction released = 1 is being used to hide products that are not released. For unreleased products, presumably released = 0.

The application doesn't implement any defenses against SQL injection attacks, so an attacker can construct an attack like:
</pre>
  ### https://insecure-website.com/products?category=Gifts'--

This results in the SQL query:

  ### SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1

The key thing here is that the double-dash sequence -- is a comment indicator in SQL, and means that the rest of the query is interpreted as a comment. This effectively removes the remainder of the query, so it no longer includes AND released = 1. This means that all products are displayed, including unreleased products.

Going further, an attacker can cause the application to display all the products in any category, including categories that they don't know about:

  ### https://insecure-website.com/products?category=Gifts'+OR+1=1--

This results in the SQL query:

  ### SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
The modified query will return all items where either the category is Gifts, or 1 is equal to 1. Since 1=1 is always true, the query will return all items.

# Subverting application logic

Consider an application that lets users log in with a username and password. If a user submits the username wiener and the password bluecheese, the application checks the credentials by performing the following SQL query:

### SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'

If the query returns the details of a user, then the login is successful. Otherwise, it is rejected.

Here, an attacker can log in as any user without a password simply by using the SQL comment sequence -- to remove the password check from the WHERE clause of the query. For example, submitting the username administrator'-- and a blank password results in the following query:

### SELECT * FROM users WHERE username = 'administrator'--' AND password = ''

This query returns the user whose username is administrator and successfully logs the attacker in as that user.

# Retrieving data from other database tables
In cases where the results of a SQL query are returned within the application's responses, an attacker can leverage a SQL injection vulnerability to retrieve data from other tables within the database. This is done using the UNION keyword, which lets you execute an additional SELECT query and append the results to the original query.

For example, if an application executes the following query containing the user input "Gifts":

### SELECT name, description FROM products WHERE category = 'Gifts'

then an attacker can submit the input:

### ' UNION SELECT username, password FROM users--

This will cause the application to return all usernames and passwords along with the names and descriptions of products.





# SQL injection UNION attacks

When an application is vulnerable to SQL injection and the results of the query are returned within the application's responses, the UNION keyword can be used to retrieve data from other tables within the database. This results in a SQL injection UNION attack.

The UNION keyword lets you execute one or more additional SELECT queries and append the results to the original query. For example:

### SELECT a, b FROM table1 UNION SELECT c, d FROM table2

This SQL query will return a single result set with two columns, containing values from columns a and b in table1 and columns c and d in table2.

For a UNION query to work, two key requirements must be met:

-> The individual queries must return the same number of columns.
-> The data types in each column must be compatible between the individual queries.
To carry out a SQL injection UNION attack, you need to ensure that your attack meets these two requirements. This generally involves figuring out:

-> How many columns are being returned from the original query?
-> Which columns returned from the original query are of a suitable data type to hold the results from the injected query?

## Determining the number of columns required in a SQL injection UNION attack
<pre>
When performing a SQL injection UNION attack, there are two effective methods to determine how many columns are being returned from the original query.

The first method involves injecting a series of ORDER BY clauses and incrementing the specified column index until an error occurs. For example, assuming the injection point is a quoted string within the WHERE clause of the original query, you would submit:

' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
etc.

This series of payloads modifies the original query to order the results by different columns in the result set. The column in an ORDER BY clause can be specified by its index, so you don't need to know the names of any columns. When the specified column index exceeds the number of actual columns in the result set, the database returns an error, such as:

The ORDER BY position number 3 is out of range of the number of items in the select list.

The application might actually return the database error in its HTTP response, or it might return a generic error, or simply return no results. Provided you can detect some difference in the application's response, you can infer how many columns are being returned from the query.

The second method involves submitting a series of UNION SELECT payloads specifying a different number of null values:

' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
etc.

If the number of nulls does not match the number of columns, the database returns an error, such as:
</pre>
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.



# Finding columns with a useful data type in a SQL injection UNION attack

The reason for performing a SQL injection UNION attack is to be able to retrieve the results from an injected query. Generally, the interesting data that you want to retrieve will be in string form, so you need to find one or more columns in the original query results whose data type is, or is compatible with, string data.

Having already determined the number of required columns, you can probe each column to test whether it can hold string data by submitting a series of UNION SELECT payloads that place a string value into each column in turn.

For example, if the query returns four columns, you would submit:

<pre>
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
If the data type of a column is not compatible with string data, the injected query will cause a database error, such as:
</pre>

### Conversion failed when converting the varchar value 'a' to data type int.

If an error does not occur, and the application's response contains some additional content including the injected string value, then the relevant column is suitable for retrieving string data.


# Using a SQL injection UNION attack to retrieve interesting data

When you have determined the number of columns returned by the original query and found which columns can hold string data, you are in a position to retrieve interesting data.

<pre>
Suppose that:

The original query returns two columns, both of which can hold string data.
The injection point is a quoted string within the WHERE clause.
The database contains a table called users with the columns username and password.
</pre>

In this situation, you can retrieve the contents of the users table by submitting the input:

### ' UNION SELECT username, password FROM users--

Of course, the crucial information needed to perform this attack is that there is a table called users with two columns called username and password. Without this information, you would be left trying to guess the names of tables and columns. In fact, all modern databases provide ways of examining the database structure, to determine what tables and columns it contains.


# Retrieving multiple values within a single column

In the preceding example, suppose instead that the query only returns a single column.

You can easily retrieve multiple values together within this single column by concatenating the values together, ideally including a suitable separator to let you distinguish the combined values. For example, on Oracle you could submit the input:

### ' UNION SELECT username || '~' || password FROM users--

This uses the double-pipe sequence || which is a string concatenation operator on Oracle. The injected query concatenates together the values of the username and password fields, separated by the ~ character.

The results from the query will let you read all of the usernames and passwords, for example:

<pre>
...
administrator~s3cure
wiener~peter
carlos~montoya
...
</pre>

# Examining the database

Following initial identification of a SQL injection vulnerability, it is generally useful to obtain some information about the database itself. This information can often pave the way for further exploitation.

You can query the version details for the database. The way that this is done depends on the database type, so you can infer the database type from whichever technique works. For example, on Oracle you can execute:

### SELECT * FROM v$version

You can also determine what database tables exist, and which columns they contain. For example, on most databases you can execute the following query to list the tables:

### SELECT * FROM information_schema.tables

# Examining the database in SQL injection attacks

When exploiting SQL injection vulnerabilities, it is often necessary to gather some information about the database itself. This includes the type and version of the database software, and the contents of the database in terms of which tables and columns it contains.

## Querying the database type and version

Different databases provide different ways of querying their version. You often need to try out different queries to find one that works, allowing you to determine both the type and version of the database software.

The queries to determine the database version for some popular database types are as follows:

<pre>
Database type	          Query
Microsoft, MySQL	  SELECT @@version
Oracle	            SELECT * FROM v$version
PostgreSQL	        SELECT version()
</pre>

For example, you could use a UNION attack with the following input:

### ' UNION SELECT @@version--

This might return output like the following, confirming that the database is Microsoft SQL Server, and the version that is being used:

<pre>
Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)
Mar 18 2018 09:11:49
Copyright (c) Microsoft Corporation
Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)
</pre>

# Listing the contents of the database

<pre>
Most database types (with the notable exception of Oracle) have a set of views called the information schema which provide information about the database.

You can query information_schema.tables to list the tables in the database:

SELECT * FROM information_schema.tables
This returns output like the following:

TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
=====================================================
MyDatabase     dbo           Products    BASE TABLE
MyDatabase     dbo           Users       BASE TABLE
MyDatabase     dbo           Feedback    BASE TABLE
This output indicates that there are three tables, called Products, Users, and Feedback.

You can then query information_schema.columns to list the columns in individual tables:

SELECT * FROM information_schema.columns WHERE table_name = 'Users'
This returns output like the following:

TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  COLUMN_NAME  DATA_TYPE
=================================================================
MyDatabase     dbo           Users       UserId       int
MyDatabase     dbo           Users       Username     varchar
MyDatabase     dbo           Users       Password     varchar
This output shows the columns in the specified table and the data type of each column.

</pre>

# Equivalent to information schema on Oracle

On Oracle, you can obtain the same information with slightly different queries.

You can list tables by querying all_tables:

  SELECT * FROM all_tables

And you can list columns by querying all_tab_columns:

  SELECT * FROM all_tab_columns WHERE table_name = 'USERS'
