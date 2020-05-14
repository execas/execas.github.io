---
layout: post
title: "SQL injection"
date: 2018-12-20
tags: [security, webapp, linux, programming, vulnerabilities]
---

Injection flaws, most commonly associated with SQL injection, are a constant feature of just about every list of top security risks, including the lastest OWASP Top Ten.

In this post we'll look at a few examples of SQL injections, and how you can get some practice legally.

## Practice Tools

### Download an example database

Using Python scripting and an example database is a good way to experiment with SQL injections.

```bash
$ git clone https://github.com/datacharmer/test_db
$ cd test_db
$ sudo systemctl start mysql
$ mysql -u root -t -p < employees.sql
Enter password:
+-----------------------------+
| INFO                        |
+-----------------------------+
| CREATING DATABASE STRUCTURE |
+-----------------------------+
+------------------------+
| INFO                   |
+------------------------+
| storage engine: InnoDB |
+------------------------+
+---------------------+
| INFO                |
+---------------------+
| LOADING departments |
+---------------------+
...
```

Next, install the MySQL Python driver:

```bash
$ pip install mysql.connector
```

### Download DVWA

Damn Vulnerable Web App (DVWA) is a great tool to experiment with SQL injection flaws and other web application vulnerabilities.

Run DVWA with Docker:

```
$ sudo systemctl start docker
$ docker run --rm -it -p 80:80 vulnerables/web-dvwa
[+] Starting mysql...
[ ok ] Starting MariaDB database server: mysqld.
[+] Starting apache
...
```

Then go to `localhost` in a web browser and log in with *admin*/*password*.


## SQL injection examples

### Example 1

A webpage allows a logged in employee to acccess their own salary information.

The database is queried with the `user_id` (set when the user authenticated) as the employee number, and a user-specified year.

```python
import mysql.connector as msc

conn = msc.connect(host = "localhost", user = "user", passwd="passwd", db="employees")
curs = conn.cursor()

user_id = 10099

curs.execute("SELECT first_name FROM employees WHERE emp_no=%s", (user_id,))
res = curs.fetchall()[0][0]
print("Welcome, {}\nDisplay salary information".format(res))
val = raw_input("From year: ")

query = ("SELECT salary,from_date FROM salaries WHERE "
            "from_date>='{}-01-01' AND emp_no={}").format(val, user_id)
print(query)
curs.execute(query)
res = curs.fetchall()
for i in res:
    print("{}: ${}".format(i[1], i[0])

conn.close()
```

### Expected usage

The user inputs 1999, and the everything works as intended:

```
Welcome, Valter
Display salary information
From year: 1999<ENTER>
1999-10-16: $93297
2000-10-15: $95842
2001-10-15: $98538
```

The user input transforms the SQL query to:

```sql
SELECT salary,from_date FROM salaries WHERE from_date>='1999-01-01' AND emp_no=10099
```

#### Malicious usage

A malicious user inputs "' AND emp_no=10998-- " to view another employee's salary information:

```
Welcome, Valter
Display salary information
From year: ' AND emp_no=10998-- <ENTER>
1996-08-07: $62861
1997-08-07: $64876
1998-08-07: $64715
1999-08-07: $66050
2000-08-06: $68437
2001-08-06: $68449
```

The user input transforms the SQL query to:

```sql
SELECT salary,from_date FROM salaries WHERE from_date>='' AND emp_no=10998-- -01-01' AND emp_no=10099
```

> "-- " (dash, dash, space) marks the start of a comment.

The malicious user would need some insight into how the database and queries are structured, or be able to do experimentation to acheive the above. An injection that can be more easily found is "'-- " (single quote, dash, dash, space) , which dumps all salaries.

#### Remediation

Validate the user input, for example by using (in Python3):

```python
val = int(input("From year: "))
if val not in range(1900, 2100):
    <fail>
```

### Example 2

A webpage allows a logged in department manager to view the salary of department employees.

```python
import mysql.connector as msc
from datetime import datetime

conn = msc.connect(host = "localhost", user = "user", passwd="passwd", db="employees")
curs = conn.cursor()

user_id = 110567
timenow = datetime.now().strftime("%Y-%m-%d")

curs.execute("SELECT first_name FROM employees WHERE emp_no=%s", (user_id,))
name = curs.fetchall()[0][0]
curs.execute("SELECT dept_name FROM departments WHERE dept_no IN (SELECT dept_no FROM dept_manager WHERE emp_no=%s AND to_date>%s)", (user_id, timenow))
dept = curs.fetchall()[0][0]

print("Welcome, {}"
      "\nAccess: {} dept."
      "\nDisplay employee salary information").format(name, dept)
val = raw_input("Employee ID: ")

query = ("SELECT first_name, last_name FROM employees "
         "WHERE emp_no = {};"
         "SELECT salary FROM salaries "
         "WHERE emp_no={} AND emp_no IN (SELECT emp_no FROM dept_emp "
         "WHERE dept_no IN (SELECT dept_no FROM dept_manager WHERE emp_no={})) "
         "GROUP BY from_date"
         ).format(val, val, user_id)

it = curs.execute(query, multi=True)
name = next(it).fetchall()
salary= next(it).fetchall()

if not salary:
    print "Employee not found in {} dept.".format(dept)
else:
    print "Name: {} {}, Salary: ${}".format(name[0][0], name[0][1], salary[-1][0])

conn.close()
```

> Note that `multi=True` is set in the curs.execute(), meaning multiple statements can be executed in one operation.

#### Expected usage

The user inputs 10099, and is informed that the employee is not found:

```
Welcome, Leon
Access: Development dept.
Display employee salary information
Employee ID: 10099<ENTER>
Employee not found in Development dept.
```

The user inputs 13627, and is displayed the salary information:

```
Welcome, Leon
Access: Development dept.
Display employee salary information
Employee ID: 13627<ENTER>
Name: Gennadi Yoshizawa, Salary: $56740
```

#### Malicious usage

A malicious user is in this example able to manipulate the first query built with input data, but is also able to execute multiple queries (*stacked queries*), which opens up a lot of interesting possibilities.

With some knowledge about SQL, we can try to leak, manipulate or destroy database data.

> Some interesting commands include `SELECT`, `INSERT`, `UPDATE`, `DROP` and `SELECT LOAD_FILE("/some/file")`.

There are, however, limitations to what can be acheived based on how the application is built and the database permissions.

In our example the output is limited due to how query results are handled and displayed, and the number of stacked queries that will be executed is also limited as a result of how the iterator returned by `curs.execute` is used.

> Interesting: The `conn.commit()` method is not used. What are the results of this?

### Example 3: DVWA

Try out SQL injection in DVWA. You can change the security settings to make the exploitation more challenging (the security level can be set to low, medium, high or impossible).

#### Code analysis

Let's do a quick code analysis focusing on the how the query string is built and used, and how results and errors are displayed.

*You should not continue reading before trying to exploit DVWA, if you want the full benefit of trial and error.*

With the *low security level* activated, the user input is directly inserted into a SQL query string.

```php
$id = $_REQUEST[ 'id' ];

$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die(...
```

> `mysqli_query()` does not support stacked queries. `mysqli_multi_query()` does.

> `$_REQUEST` is an associative array (`key, value` pairs) that contains the contents of `$_GET`, `$_POST` and `_$COOKIE`. 

On error (`die`), a verbose message is displayed:

```
You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near '';''' at line 1
```

On success, all fetched rows are output as html:

```php
while( $row = mysqli_fetch_assoc( $result ) ) {
	// Get values
	$first = $row["first_name"];
	$last  = $row["last_name"];

	// Feedback for end user
	$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
}
```

With the *medium security level* activated, we got the following:

```php
$id = $_POST[ 'id' ];

$id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

$query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
$result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die(...
```

> `_$POST` is an associative array containing variables passed via the `HTTP POST` method (which transfers information in HTTP headers).

A new addition here is `mysqli_real_escape_string()`, which will return an escaped string (or False on error). It simply prepends backslashes to `NUL` (`\x00`), `\n`, `\r`, `\`, `'`, `"` and `SUB` (`\x1a` -- `Ctrl-z`).

The output code and the die message are the same as for *low*.

With the *high security level* activated, the `id` variable is now set in `session-input.php`:

```php
if( isset( $_POST[ 'id' ] ) ) {
	$_SESSION[ 'id' ] =  $_POST[ 'id' ];
```

But it's still used as part of the query string without validation.

```php
$id = $_SESSION[ 'id' ];

$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
$result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>Something went wrong.</pre>' );k
```

The new addition here is `LIMIT 1` which will only select one (the first) record.

On error, a simple "Something went wrong." message is displayed. The results are displayed like before.

With the *impossible security level* activated, the code now contains multiple checks:

```
<?php

if( isset( $_GET[ 'Submit' ] ) ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

	// Get input
	$id = $_GET[ 'id' ];

	// Was a number entered?
	if(is_numeric( $id )) {
		// Check the database
		$data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );
		$data->bindParam( ':id', $id, PDO::PARAM_INT );
		$data->execute();
		$row = $data->fetch();

		// Make sure only 1 result is returned
		if( $data->rowCount() == 1 ) {
			// Get values
			$first = $row[ 'first_name' ];
			$last  = $row[ 'last_name' ];

			// Feedback for end user
			$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
		}
	}
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

> `CSRF` will be covered in another post.

> `_$GET` is an associative array containing variables passed via URL parameters ("`.php?v1=1&v2=2`").

Nothing is done unless `id` is a number, and `id` is inserted into the query string using `bindParam()` with data type `int`. The SQL query itself uses `LIMIT 1`, and output is only displayed if 1 row was returned for the SQL query.



## SQL and database Security

As we saw in the above examples, input validation and sanitization are - like for many other vulnerabilites - key. Malicious input data can leak, manipulate or destroy database data.

There are multiple security measures that will help to secure databases. In the next post we'll cover SQL and database security in detail.
