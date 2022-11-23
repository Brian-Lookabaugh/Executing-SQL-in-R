# Executing SQL in R

A promising push in the world of data analytics and data science is the push to homogenize the tools that are used into a common interface. Given my background in R, the prospect of working with SQL *completely inside of RStudio* has always been very appealing to me. This blog serves as a very *brief* and *introductory* review of how to execute SQL in R along with a couple of other related features. While this brief guide is not intended to serve as an introduction to SQL syntax, it is my intent that it can be shared with those who are new to SQL with R. Further, like all of my blog posts, there is a strong possibility I personally return to this at some point when I need a refresher on syntax and packages.

To begin, we will load the packages that will be utilized in this blog.

```{r warning=FALSE, message=FALSE}
pacman::p_load(
  "dplyr", ## Data Manipulation in R
  "sqldf", ## Running SQL Queries in R
  "dbplyr", ## Translating dplyr Syntax to SQL Syntax
  "DBI", ## Connecting to a Database
  "odbc", ## Connecting to a Database
  "tidyqyery", ## Translating SQL Syntax to dplyr Syntax
  install = FALSE
)
```

### Setting Up Databases

In practice, executing SQL in R requires the connection to a pre-existing SQL database. For the purpose of this blog, however, we will just be using a temporary database stored in a local RStudio session. We will store this database as an object call `con`.

```{r}
con <- DBI::dbConnect(RSQLite::SQLite(), ":memory:")
```

For practical reasons, the syntax above will not be sufficient. Each connection will look different, dependent on various circumstances (the type of relation database management system (RDBMS) being used, log-in information, etc.), so the following example is just that; an example using completely made-up information. However, it does serve as a template for real information to be plugged into.

```{r eval=FALSE}
con <- dbConnect(odbc(),
                 Driver = ,
                 Server = ,
                 Database = ,
                 UID = ,
                 PWD = ,
                 Port = )
```

Returning to the database we created, it is empty and has no data stored in it. To keep things simple, we are going to load the `mtcars` data set. We first begin by loading the data into RStudio. The second line of code copies this data set into the local database that we created. Now that we have copied this data into the local database, we can remove the `mtcars` data set from the local environment.

```{r}
data("mtcars")

dbWriteTable(conn = con,
             name = "mtcars",
             value = mtcars)
             
rm(mtcars)
```

### Running a SQL Query in R

Now that we have the `mtcars` data in our database, we can run a SQL query to retrieve information from this data. Using the `dbGetQuery` command, we can execute SQL syntax to return the desired information. Here, we are writing a query to return a table which tells us the average miles per gallon for automatic vehicles grouped by the number of cylinders the vehicle has and ordered by miles per gallon from the highest to lowest values.

```{r}
query_1 <- dbGetQuery(con,
  'SELECT ROUND(AVG(mpg)) as avg_mpg, cyl
   FROM mtcars
   WHERE am = 1
   GROUP BY cyl
   ORDER BY avg_mpg DESC;'
)

tibble(query_1)
```
```{r}
# A tibble: 3 × 2
  avg_mpg   cyl
    <dbl> <dbl>
1      28     4
2      21     6
3      15     8
```

In contrast, if you wanted to execute a query on a data frame object instead of pulling from a database, you can use `sqldf`.

```{r}
query_2 <- sqldf(
  'SELECT ROUND(AVG(mpg)) as avg_mpg, cyl
   FROM mtcars
   WHERE am = 1
   GROUP BY cyl
   ORDER BY avg_mpg DESC;'
)

tibble(query_2)
```
```{r}
# A tibble: 3 × 2
  avg_mpg   cyl
    <dbl> <dbl>
1      28     4
2      21     6
3      15     8
```

### Running a SQL Chunk in RMarkdown/Quarto

We can conveniently execute a SQL query in R without relying on a specific command like `dbGetQuery`. Using RMarkdown or Quarto, we can specify a SQL code chunk. Within the code chunk, you will need to specify the connection (`con` in our case) and, optionally, the object that the results of the query will be stored in. In the output below, you would begin the code chunk with `{sql, connection = con, output.var = "query_2"}`.

```{sql, connection = con, output.var = "query_3"}
SELECT
  ROUND(AVG(mpg)) AS avg_mpg,
  cyl
FROM mtcars
WHERE am = 1
GROUP BY cyl
ORDER BY avg_mpg DESC;
```

```{r}
tibble(query_3)
```
```{r}
# A tibble: 3 × 2
  avg_mpg   cyl
    <dbl> <dbl>
1      28     4
2      21     6
3      15     8
```

Note that if you are going to be using SQL chunks frequently, it is worth specifying the default connection for SQL chunks as demonstrated below.

```{r eval=FALSE}
knitr::opts_chunk$set(connection = "con")
```

## Translating dplyr Syntax to SQL Syntax and Vice Versa

Another very helpful tool that bridges the gap between SQL and dplyr syntax is the `show_query` command. Personally, I found this tool incredibly valuable when learning SQL because of my background in R. Essentially, what this tool does is translate dplyr syntax into SQL syntax. In the opposite direction, through the `tidyquery` package, we also have the capability to the exact opposite and translate SQL syntax into dplyr syntax. Below demonstrates the functionality of these two commands for the same query. First, translating dplyr syntax to SQL syntax:

```{r}
tbl(con, "mtcars") %>%
  filter(am == 1) %>%
  group_by(cyl) %>%
  summarise(avg_mpg = round(mean(mpg))) %>%
  ungroup() %>%
  arrange(dplyr::desc(avg_mpg)) %>%
  show_query()
```
```{r}
<SQL>
SELECT `cyl`, ROUND(AVG(`mpg`), 0) AS `avg_mpg`
FROM `mtcars`
WHERE (`am` = 1.0)
GROUP BY `cyl`
ORDER BY `avg_mpg` DESC
```

Now, we will do the opposite

```{r}
show_dplyr(
  "SELECT
    ROUND(AVG(mpg)) AS avg_mpg,
    cyl
   FROM mtcars
   WHERE am = 1
   GROUP BY cyl
   ORDER BY avg_mpg DESC;"
)
```
```{r}
mtcars %>%
  filter(am == 1) %>%
  group_by(cyl) %>%
  summarise(avg_mpg = round(mean(mpg, na.rm = TRUE))) %>%
  ungroup() %>%
  arrange(dplyr::desc(avg_mpg))
```

Obviously, as one's knowledge in both SQL and R increases, the further capabilities of executing SQL in R can be explored. My hope is that this serves as a helpful introductory for those seeking to integrate data science tools together.
