---
title: Programming with Databases - R with dplyr
teaching: 30
exercises: 15
questions:
- "How can I access databases from programs written in R with dplyr?"
objectives:
- "Write short programs that execute SQL queries."
- "Trace the execution of a program that contains an SQL query."
- "Explain why most database applications are written in a general-purpose language rather than in SQL."
keypoints:
- "Data analysis languages have libraries for accessing databases."
- "To connect to a database, a program must use a library specific to that database manager."
- "R's libraries can be used to directly query or read from a database."
- "Programs can read query results in batches or all at once."
- "Queries should be written using parameter substitution, not string formatting."
- "R has multiple helper functions to make working with databases easier."
---

To close,
let's have a look at how to access a database from
a data analysis language like R.
Other languages use almost exactly the same model:
library and function names may differ,
but the concepts are the same.

It's our job to make sure that SQL is properly formatted;
if it isn't,
or if something goes wrong when it is being executed,
the database will report an error.



Here's a short R program that selects latitudes and longitudes
from an SQLite database stored in a file called `survey.db`:


~~~
library(dplyr)
library(RSQLite)
connection <- src_sqlite("data/survey.db")
results <- tbl(connection, "Site") %>%
  select(lat, long) %>%
  collect
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}



~~~
results
~~~
{: .r}



~~~
Error in eval(expr, envir, enclos): object 'results' not found
~~~
{: .error}

Let's break this down into pieces to understand what is done.


~~~
library(dplyr)
library(RSQLite)
connection <- src_sqlite("data/survey.db")
~~~
{: .r}

The program starts by attaching the `dplyr` and `RSQLite` packages.
If we were connecting to MySQL, PostgreSQL, or some other database,
we would use a slightly different command,
but all of them provide the same functions,
so that the rest of our program does not have to change
if we switch from one database to another.

Line 2 establishes a connection to the database.
Since we're using SQLite,
all we need to specify is the name of the database file.
Other systems may require us to provide a username and password as well.


~~~
preview <- tbl(connection, "Site") %>%
  select(lat, long)
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}



~~~
preview
~~~
{: .r}



~~~
Error in eval(expr, envir, enclos): object 'preview' not found
~~~
{: .error}

On lines 3-4, we connect to the table `Site` in the database
and select the columns we desire. This is a `preview`, as we
are only looking into the database right now. Notice that the 
print of the preview shows `??` for unknown number of rows,
even though we can see there are only 3.


~~~
results <- preview %>%
  collect
~~~
{: .r}



~~~
Error in eval(expr, envir, enclos): object 'preview' not found
~~~
{: .error}



~~~
results
~~~
{: .r}



~~~
Error in eval(expr, envir, enclos): object 'results' not found
~~~
{: .error}

The last line `collect`s the table locally in R, and `results` 
is a dataframe with one row for each entry and one column for 
each column in the database.
The `dplyr` package differs from `RSQLite` in that there is
no need to manually close the connection. This is done behind
the scenes by `dplyr` helper functions.
It's normal to create one `connection` that is used for the
lifetime of the program.

The `dplyr` verbs `select()`, `filter()`, `arrange()`, `mutate()`,
and `summarise()` are translated into SQL, and can be used to
organize a table before the `collect` step. Other commands to `join`
tables together are useful, but are not covered here.

Queries in real applications will often depend on values provided by users.
For example,
this function takes a user's ID as a parameter and returns their name.
Here are three ways to do this in `dplyr`, the wrong way, a better way,
and the preferred method. The first to ways use SQL verbs through the `sql()`
and `build_sql()` functions in `dplyr`.


~~~
getName_wrong_way <- function(connection, personID) {
  query <- paste0("SELECT personal || ' ' || family FROM Person WHERE id =='",
                  personID, "'")
  tbl(connection, sql(query)) %>%
    collect
}
paste("full name for dyer:", getName_wrong_way(connection, 'dyer'))
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}


~~~
getName_better_way <- function(connection, personID) {
  query <- build_sql("SELECT personal || ' ' || family FROM Person WHERE id ==",
                      personID)
  tbl(connection, query) %>%
    collect
}
paste("full name for dyer:", getName_better_way(connection, 'dyer'))
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}


~~~
getName <- function(connection, personID) {
  tbl(connection, "Person") %>%
    filter(id == personID) %>%
    select(personal, family) %>%
    collect %>%
    mutate(full_name = paste(personal, family)) %>%
    select(full_name)
}
paste("full name for dyer:", getName(connection, 'dyer'))
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}

We use string concatenation on the first line of this function
to construct a query containing the user ID we have been given.
This seems simple enough,
but what happens if someone gives us this string as input?

~~~ 
dyer'; DROP TABLE Survey; SELECT '
~~~
{: .sql}

It looks like there's garbage after the user's ID,
but it is very carefully chosen garbage.
If we insert this string into our query,
the result is:

~~~ 
SELECT personal || ' ' || family FROM Person WHERE id='dyer'; DROP TABLE Survey; SELECT '';
~~~
{: .sql}

If we execute this,
it will erase one of the tables in our database.

This is called an [SQL injection attack](reference.html#sql-injection-attack),
and it has been used to attack thousands of programs over the years.
In particular,
many web sites that take data from users insert values directly into queries
without checking them carefully first.
A very [relevant XKCD](https://xkcd.com/327/) that explains the 
dangers of using raw input in queries a little more succinctly:

![relevant XKCD](https://imgs.xkcd.com/comics/exploits_of_a_mom.png) 

Since an unscrupulous parent might try to smuggle commands into our queries in many different ways,
the safest way to deal with this threat is
to replace characters like quotes with their escaped equivalents,
so that we can safely put whatever the user gives us inside a string.
We can do this by using `build_sql()` as shown above.
The key changes are in the query string, which no longer has an open quote,
and the use of `build_sql()` instead of `sql()`.

There are times when explicit SQL is useful, but most of the time
this can be avoided using `dplyr` verbs.

> ## Filling a Table vs. Printing Values 
>
> Write an R program that creates a new database in a file called
> `original.db` containing a single table called `Pressure`, with a
> single field called `reading`, and inserts 100,000 random numbers
> between 10.0 and 25.0.  How long does it take this program to run?
> How long does it take to run a program that simply writes those
> random numbers to a file?
>
> > ## Solution
> >
> > 
> > ~~~
> > original_db <- src_sqlite("original.db", create = TRUE)
> > Pressure <- data_frame(reading = runif(100000, 10, 25))
> > system.time(copy_to(original_db, Pressure))
> > system.time(write.csv(Pressure, file="Pressure.csv"))
> > ~~~
> > {: .r}
> {: .solution}
{: .challenge}

> ## Filtering in SQL vs. Filtering in R
>
> Write an R program that creates a new database called
> `backup.db` with the same structure as `original.db` and copies all
> the values greater than 20.0 from `original.db` to `backup.db`.
> Which is faster: filtering values in the query, or reading
> everything into memory and filtering in R?
>
> > ## Solution
> >
> > 
> > ~~~
> > backup_db <- src_sqlite("backup.db", create = TRUE)
> > system.time(copy_to(backup_db, tbl(original_db, "Pressure") %>% filter(reading > 20) %>% collect, name = "Pressure_pre"))
> > system.time(copy_to(backup_db, tbl(original_db, "Pressure") %>% collect %>% filter(reading > 20), name = "Pressure_post"))
> > ~~~
> > {: .r}
> {: .solution}
{: .challenge}

## Database helper functions in R

R's database interface packages (like `dplyr` and `RSQLite`) all share 
a common set of helper functions useful for exploring databases and 
reading/writing entire tables at once. 
The helpers for `dplyr` are documented in the help page `help("backend_db")`.
However, if we are doing standard operations, it is easier to examine
the database and tables directly rather than use these helpers.

To view all tables in a database, we can use `db_list_tables()`


~~~
db_list_tables(connection$con)
~~~
{: .r}



~~~
Error in UseMethod("db_list_tables"): no applicable method for 'db_list_tables' applied to an object of class "SQLiteConnection"
~~~
{: .error}

Notice the explicit use of the element `con` from `connection`.
Much easier to get this information from `connection` itself:


~~~
connection
~~~
{: .r}



~~~
Error in UseMethod("db_list_tables"): no applicable method for 'db_list_tables' applied to an object of class "SQLiteConnection"
~~~
{: .error}

To read an entire table as a dataframe, use `tbl()`:


~~~
tbl(connection, "Person")
~~~
{: .r}



~~~
Error in UseMethod("db_query_fields"): no applicable method for 'db_query_fields' applied to an object of class "SQLiteConnection"
~~~
{: .error}

This shows the `head` of the table. Remember to use `collect`
to actually load the table into R.

Finally to write an entire table to a database, you can use `copy_to()`. 
In this example we will write R's built-in `iris` dataset as a table in `survey.db`.


~~~
copy_to(connection, iris)
~~~
{: .r}



~~~
Error in UseMethod("db_has_table"): no applicable method for 'db_has_table' applied to an object of class "SQLiteConnection"
~~~
{: .error}

We can verify it is in the database:


~~~
connection
~~~
{: .r}



~~~
Error in UseMethod("db_list_tables"): no applicable method for 'db_list_tables' applied to an object of class "SQLiteConnection"
~~~
{: .error}

There is no need to close the database connection when using `dplyr`
as that is done automatically.
