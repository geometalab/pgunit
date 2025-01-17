# PGUnit - Lightweight Unit Test Framework for PL/pgSQL (PostgreSQL Extension)

This is a fork of https://github.com/adrianandrei-ca/pgunit. Many thanks to the developers.

The purpose of this suite of stored procedures in PL/pgSQL is to allow a user to run unit tests on PL/pgSQL programs.

The testing is based on a specific naming convention that allows automatic grouping of tests, setup, tear-downs, pre- and post- conditions.

Each unit test procedure name should have "test_case_" prefix in order to be identified as an unit test. Here is the comprehensive list of prefixes for all types:
- "test_case_": identifies an unit test procedure
- "test_precondition_": identifies a test precondition function
- "test_postcondition_": identifies a test postcondition  function
- "test_setup_": identifies a test setup procedure
- "test_teardown_": identifies a test tear down procedure.

For each test case the following 3 transactions are executed:
1. Setup transaction: the setup procedure is searched based on the test name. If one is found it is executed in an autonomous transaction
2. Unit test transaction: the pre and post condition functions are searched based on the test name; if they are found the autonomous transaction will be: if the precondition is true (default if one is not found) the unit test is ran, then the postcondition function is evaluated (true if one is not found). If any condition returns false the test is failed
3. Tear down transaction: if a tear down procedure is found it is executed in an autonomous transaction indepedent of the unit test result.

A unit test execution can have 3 results: successful if the condition functions are true and the unit test procedure doesn't throw an exception, failed if there is an action exception triggered by a condition function or an assertion, and finally erroneous if any other exeception is triggered by any of the code above.

## Tests logical grouping

The tests can be grouped based on their name structure. The underscore character is used to separate the test name into a hierarchical structure so that you can share the setup, tear down, or the preconditions across several unit tests. For instance the tests 'test_case_finance_audit_x' and 'test_case_finance_accounting_y' can share the setup procedure called 'test_setup_finance', the teardown procedure 'test_teardown_finance' as well as the precondition 'test_precondition_finance' function as they share a common prefix.

If, let's say, the audit tests have a specific precondition function, you can define a new 'test_precondition_finance_audit' function and that will be shared accross all unit test procedures with prefix 'test_case_finance_audit_' and override the common 'test_precondition_finance' function.

Using the built-in grouping mechanism you can re-use supporting code such as the data setup and tear down across unit tests . There is little if any gain if the save data setup is shared across multiple tests and they will have to be present in test-specific setup function while having the exact same content.

## Running one or more tests

To run the entire test suite the 'pgunit.run_all' stored procedure needs to be used:
```sql
select * from pgunit.run_all();
```
You can pick one or an entire group of tests based on their prefix using the 'pgunit.run_suite' stored procedure:
```sql
select * from pgunit.run_suite('finance');
```

The statement above will pick up all unit tests starting with the 'test_case_finance' prefix together with the associated support functions and procedures.

The select statement returns the type 'pgunit.results', which is:
```sql
create type pgunit.results as (
  test_name varchar, 
  successful boolean, 
  failed boolean, 
  erroneous boolean, 
  error_message varchar,
  duration interval);
```
## Installing PGUnit

Simply download this repository or clone it directly with git
``` sh
git clone https://github.com/geometalab/pgunit.git
```
### Linux

PGUnit is a Postgresql extension. In order to install it, move `pgunit--0.1.0.sql` and `pgunit.control` into your extension folder. On Unix, this can be done using `make`:

If not already, make the Makefile executable
``` sh
chmod u+x Makefile 
```

``` sh
sudo make install
```

### Windows

Copy the `pgunit.control` and the `pgunit--0.1.0.sql` files into `C:\Program Files\Postgresql\VERSION\share\extension`, where `VERSION` is your postgresql version (e.g. `14`).

### MacOS

You can use `make` in the cloned repo to install the extension to the correct location:

```sh
make install
```

## Setting up PGUnit

After you successfully installed PGUnit like explained above, load PGUnit (as Superuser) using:
``` sql
create extension pgunit;
```
Please note, Postgresql Extensions have to be loaded by a Super-User.

### Test as Non-Superuser
If you want to test your pgunit functions as a non-Superuser, simply grant access to special user PUBLIC.
``` sql
grant usage on schema pgunit to PUBLIC;
```

## Assertion procedures

| Procedure | Description |
| --- | --- |
|`pgunit.assertTrue(message VARCHAR, condition BOOLEAN) returns void`|If condition is false it throws an exception with the given message|
|`pgunit.assertTrue(condition BOOLEAN) returns void` |Similar to `assertTrue` above but with no user message|
|`pgunit.assertFalse(message VARCHAR, condition BOOLEAN) returns void`|If condition is true it throws an exception with the given message|
|`pgunit.assertFalse(condition BOOLEAN) returns void` |Similar to `assertFalse` above but with no user message|
|`pgunit.assertNotNull(message VARCHAR, data ANYELEMENT) returns void`|If the data is null an exception is thrown with the message provided|
|`pgunit.assertNull(message VARCHAR, data ANYELEMENT) returns void`|If the data is not null an exception is thrown with the message provided|
|`pgunit.fail(message VARCHAR) returns void`|If reached, the test fails with the message provided|

(Since Postgresql version 9.5 there is also an [ASSERT statement](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html#PLPGSQL-STATEMENTS-ASSERT). Currently, using this is unsupported by this library.)

## Examples

We tested PGUnit with PGUnit itself, if you need more examples, please check the meta.sql, where you can find this tests.

Test case that checks if an application user is created by a stored procedure
- the user id is returned if user id 1 is a parent
- the id is larger than a thresold
```sql
create or replace function test_case_user_create_1() returns void as $$
declare
  id BIGINT;
begin
  SELECT customer.createUser(1, 100) INTO id;
  perform pgunit.assertNotNull('user not created', id);
  perform pgunit.assertTrue('user id range improper', id >= 10000);
end;
$$ language plpgsql;
```
A precondition function for this test may be one checking for user id 1 being present in the database
```sql
create or replace function test_precondition_user() returns boolean as $$
declare
   id BIGINT;
begin
  SELECT user_id INTO id FROM customer.user WHERE user_id=1;
  RETURN id IS NOT NULL AND (id = 1);
end;
$$ language plpgsql;
```
The precondition above will be shared on all 'user' tests unless one with a more specific name is created.

## Troubleshooting

### Lock issues
Although the unit tests are run in autonomous transactions it is possible to run into lock issues and have the select statements above hanging. In this case have a new connection on the same database and issue the statement below to stop all locking sessions:
`select * from pgunit.terminate('my_db_name');`

In order to find out which test is at issue you should run the suite one test at the time. The procedure above is not specific to PGUnit and can be used in general as well; it will terminate all locking sessions.

### On Fedora

If your postgresql-server-devel doesn't support PGXS, you have to install the following:

``` sh
sudo dnf install postgresql-contrib
```
---

# Copyright and License

Copyright (c) 2016 Adrian Andrei. Some rights reserved.

Permission to use, copy, modify, and distribute this software and its documentation for any purpose, without fee, and without a written agreement is hereby granted, provided that the above copyright notice and this paragraph and the following two paragraphs appear in all copies.

IN NO EVENT SHALL ADRIAN ANDREI BE LIABLE TO ANY PARTY FOR DIRECT, INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES, INCLUDING LOST PROFITS, ARISING OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION, EVEN IF ADRIAN ANDREI HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

ADRIAN ANDREI SPECIFICALLY DISCLAIMS ANY WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE. THE SOFTWARE PROVIDED HEREUNDER IS ON AN "AS IS" BASIS, AND ADRIAN ANDREI HAS NO OBLIGATIONS TO PROVIDE MAINTENANCE, SUPPORT, UPDATES, ENHANCEMENTS, OR MODIFICATIONS.
