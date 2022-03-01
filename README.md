# SpyQL

SQL with Python in the middle

[![https://pypi.python.org/pypi/spyql](https://img.shields.io/pypi/v/spyql.svg)](https://pypi.org/project/spyql/)
[![https://travis-ci.com/dcmoura/spyql](https://img.shields.io/travis/dcmoura/spyql.svg)](https://pypi.org/project/spyql/)
[![https://spyql.readthedocs.io/en/latest/?version=latest](https://readthedocs.org/projects/spyql/badge/?version=latest)](https://spyql.readthedocs.io/en/latest/)
[![codecov](https://codecov.io/gh/dcmoura/spyql/branch/master/graph/badge.svg?token=5C7I7LG814)](https://codecov.io/gh/dcmoura/spyql)
[![code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![license: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)


## Concept

SpyQL is a query language that combines:

* the simplicity and structure of SQL
* with the power and readability of Python

```sql
SELECT
    date.fromtimestamp(purchase_ts) AS purchase_date,
    price * quantity AS total
FROM csv
WHERE department.upper() == 'IT'
TO json
```

SQL provides the structure of the query, while Python is used to define expressions, bringing along a vast ecosystem of packages.


## SpyQL command-line tool

[**Demo video**](https://vimeo.com/danielcmoura/spyqldemo)

With the SpyQL command-line tool you can make SQL-like SELECTs powered by Python on top of text data (e.g. CSV and JSON). Data can come from files but also from data streams, such as as Kafka, or from databases such as PostgreSQL. Basically, data can come from any command that outputs text :-). More, data can be generated by a Python iterator! Take a look at the examples section to see how to query parquet, process API calls, transverse directories of zipped JSONs, among many other things.

SpyQL also allows you to easily convert between text data formats:

* `FROM`: CSV, JSON, TEXT and Python iterators (YES, you can use a list comprehension as the data source)

* `TO`: CSV, JSON, SQL (INSERT statements), pretty terminal printing, and terminal plotting.

The JSON format is [JSON lines](https://jsonlines.org), where each line has a valid JSON object or array. Piping with [jq](https://stedolan.github.io/jq/) allows SpyQL to handle any JSON input (more on the examples section).

You can leverage command line tools to process other file types like Parquet and XML  (more on the examples section).

### Installation

To install SpyQL, run this command in your terminal:

```sh
pip install spyql
```

### Hello world

To test your installation run in the terminal:
```sh
spyql "SELECT 'Hello world' as Message TO pretty"
```
Output:
```
Message
-----------
Hello world
```

Try replacing the output format by json and csv, and try adding more columns. e.g. run in the terminal:

```sh
spyql "SELECT 'Hello world' as message, 1+2 as three TO json"
```
Output:
```json
{"message": "Hello world", "three": 3}
```

## Principles

Right now, the focus is on building a command-line tool that follows these core principles:

* **Simple**: simple to use with a straightforward implementation
* **Familiar**: you should feel at home if you are acquainted with SQL and Python
* **Light**: small memory footprint that allows you to process large data that fit into your machine
* **Useful**: it should make your life easier, filling a gap in the eco-system


## Syntax

```sql
[ IMPORT python_module [ AS identifier ] [, ...] ]
SELECT [ DISTINCT | PARTIALS ] 
    [ * | python_expression [ AS output_column_name ] [, ...] ]
    [ FROM csv | spy | text | python_expression | json [ EXPLODE path ] ]
    [ WHERE python_expression [ [NOT] LIKE string] ]
    [ GROUP BY output_column_number | python_expression  [, ...] ]
    [ ORDER BY output_column_number | python_expression
        [ ASC | DESC ] [ NULLS { FIRST | LAST } ] [, ...] ]
    [ LIMIT row_count ]
    [ OFFSET num_rows_to_skip ]
    [ TO csv | json | spy | sql | pretty | plot ]
```


## Notable differences to SQL

In SpyQL:

* there is guarantee that the order of the output rows is the same as in the input (if no reordering is done)
* the `AS` keyword must precede a column alias definition (it is not optional as in SQL)
* you can always access the nth input column by using the default column names `colN` (e.g. `col1` for the first column)
* currently only a small subset of SQL is supported, namely `SELECT` statements without: sub-queries, joins, set operations, etc (check the [Syntax](#syntax) section)
* sub-queries are achieved by piping (see the [Command line examples](#command line examples)
 section)
* aggregation functions have the suffix `_agg` to avoid conflicts with python's built-in functions:

| Operation | PostgreSQL | SpyQL |
| --------- | ---------- | ----- |
| Sum all values of a column | `SELECT sum(col_name)` | `SELECT sum_agg(col_name)` | 
| Sum an array | `SELECT sum(a) FROM (SELECT unnest(array[1,2,3]) AS a) AS t` | `SELECT sum([1,2,3])` |


* expressions are pure Python:

| SQL | SpySQL |
| ------------- | ------------- |
| `x = y` | `x == y` |
| `x BETWEEN a AND b`  |  `a <= x <= b` |
| `CAST(x AS INTEGER)` | `int(x)` |
| `CASE WHEN x > 0 THEN 1 ELSE -1 END` | `1 if x > 0 else -1` |
| `upper('hello')` | `'hello'.upper()` |


## Notable differences to Python

### Additional syntax

We added additional syntax for making querying easier:

| Python | SpySQL shortcut| Purpose |
| ------ | -------------- | ------- |
| `json['hello']['planet earth']` | `json->hello->'planet earth'` | Easy access of elements in  dicts (e.g. JSONs) |


### NULL datatype

Python's `None` generates exceptions when making operations on missing data, breaking query execution (e.g. `None + 1` throws a `TypeError`). To overcome this, we created a `NULL` type that has the same behavior as in SQL (e.g. `NULL + 1` returns `NULL`), allowing for queries to continue processing data.

|Operation | Native Python throws | SpySQL returns | SpySQL warning |
| ------------- | ------------- | ------------- | ------------- |
| `NULL + 1` | `NameError` | `NULL` | |
| `a_dict['inexistent_key']` | `KeyError` | `NULL` | yes |
| `int('')`   |  `ValueError` | `NULL` | yes |
| `int('abc')` | `ValueError` | `NULL` | yes |

The above dictionary key access only returns `NULL` if the dict is an instance of `NullSafeDict`. SpyQL adds `NullSafeDict`, which extends python's native `dict`. JSONs are automatically loaded as `NullSafeDict`. Unless you are creating dictionaries on the fly you do not need to worry about this.

## Importing python modules and user-defined functions

By default, spyql do some commonly used imports:
- everything from the `math` module
- `datetime`, `date` and `timezone` from the `datetime` module
- the `re` module

SpyQL queries support a single import statement at the beginning of the query where several modules can be imported (e.g. `IMPORT numpy AS np, sys SELECT ...`). Note that the python syntax `from module import identifier` is not supported in queries.

In addition, you can create a python file that is loaded before executing queries. Here you can define imports, functions, variables, etc using regular python code. Everything defined in this file is available to all your spyql queries. The file should be located at `XDG_CONFIG_HOME/spyql/init.py`. If the environment variable `XDG_CONFIG_HOME` is not defined, it defaults to `HOME/.config` (e.g. `/Users/janedoe/.config/spyql/init.py`).




## Example queries

You can run the following example queries in the terminal:
`spyql "the_query" < a_data_file`

Example data files are not provided on most cases.

### Query a CSV (and print a pretty table)

```sql
SELECT a_col_name, 'positive' if col2 >= 0 else 'negative' AS sign
FROM csv
TO pretty
```

### Convert CSV to a flat JSON

```sql
SELECT * FROM csv TO json
```

### Convert from CSV to a hierarchical JSON

```sql
SELECT {'client': {'id': col1, 'name': col2}, 'price': 120.40}
FROM csv TO json
```

or

```sql
SELECT {'id': col1, 'name': col2} AS client, 120.40 AS price
FROM csv TO json
```

### JSON to CSV, filtering out NULLs

```sql
SELECT json->client->id AS id, json->client->name AS name, json->price AS price
FROM json
WHERE json->client->name is not NULL
TO csv
```

### Explode JSON to CSV

```sql
SELECT json->invoice_num AS id, json->items->name AS name, json-items->price AS price
FROM json
EXPLODE json->items
TO csv
```

Sample input:

```json
{"invoice_num" : 1028, "items": [{"name": "tomatoes", "price": 1.5}, {"name": "bananas", "price": 2.0}]}
{"invoice_num" : 1029, "items": [{"name": "peaches", "price": 3.12}]}
```

Output:

```
id, name, price
1028, tomatoes, 1.5
1028, bananas, 2.0
1029, peaches, 3.12
```


### Python iterator/list/comprehension to JSON

```sql
SELECT 10 * cos(col1 * ((pi * 4) / 90)
FROM range(80)
TO json
```

or

```sql
SELECT col1
FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
TO json
```

### Importing python modules

Here we import `hashlib` to calculate a md5 hash for each input line.
Before running this example you need to install the `hashlib` package (`pip install hashlib`).

```sql
IMPORT hashlib as hl
SELECT hl.md5(col1.encode('utf-8')).hexdigest()
FROM text
```

### Getting the top 5 records

```sql
SELECT int(score) AS score, player_name
FROM csv
ORDER BY 1 DESC NULLS LAST, score_date
LIMIT 5
```

### Aggregations

Totals by player, alphabetically ordered.

```sql
SELECT json->player_name, sum_agg(json->score) AS total_score
FROM json
GROUP BY 1
ORDER BY 1
```

### Partial aggregations

Calculating the cumulative sum of a variable using the `PARTIALS` modifier. Also demoing the lag aggregator. 

```sql
SELECT PARTIALS 
    json->new_entries, 
    sum_agg(json->new_entries) AS cum_new_entries,
    lag(json->new_entries) AS prev_entries
FROM json
TO json
```
Sample input:

```json
{"new_entries" : 10}
{"new_entries" : 5}
{"new_entries" : 25}
{"new_entries" : null}
{}
{"new_entries" : 100}
```

Output:

```json
{"new_entries" : 10,   "cum_new_entries" : 10,  "prev_entries": null}
{"new_entries" : 5,    "cum_new_entries" : 15,  "prev_entries": 10}
{"new_entries" : 25,   "cum_new_entries" : 40,  "prev_entries": 5}
{"new_entries" : null, "cum_new_entries" : 40,  "prev_entries": 25}
{"new_entries" : null, "cum_new_entries" : 40,  "prev_entries": null}
{"new_entries" : 100,  "cum_new_entries" : 140, "prev_entries": null}
```

If `PARTIALS`was omitted the result would be equivalent to the last output row. 

### Distinct rows

```sql
SELECT DISTINCT *
FROM csv
```


## Command line examples

To run the following examples, type `Ctrl-x Ctrl-e` on you terminal. This will open your default editor (emacs/vim). Paste the code of one of the examples, save and exit.

### Queries on Parquet with directories

Here, `find` transverses a directory and executes `parquet-tools` for each parquet file, dumping each file to json format. `jq -c` makes sure that the output has 1 json per line before handing over to spyql. This is far from being an efficient way to query parquet files, but it might be a handy option if you need to do a quick inspection.

```sh
find /the/directory -name "*.parquet" -exec parquet-tools cat --json {} \; |
jq -c |
spyql "
	SELECT json->a_field, json->a_num_field * 2 + 1
	FROM json
"
```

### Querying multiple json.gz files

```sh
gzcat *.json.gz |
jq -c |
spyql "
	SELECT json->a_field, json->a_num_field * 2 + 1
	FROM json
"
```

### Querying YAML / XML / TOML files

[yq](https://kislyuk.github.io/yq/#) converts yaml, xml and toml files to json, allowing to easily query any of these with spyql.

```sh
cat file.yaml | yq -c | spyql "SELECT json->a_field FROM json"
```
```sh
cat file.xml | xq -c | spyql "SELECT json->a_field FROM json"
```
```sh
cat file.toml | tomlq -c | spyql "SELECT json->a_field FROM json"
```


### Kafka to PostegreSQL pipeline

Read data from a kafka topic and write to postgres table name `customer`.

```sh
kafkacat -b the.broker.com -t the.topic |
spyql -Otable=customer -Ochunk_size=1 --unbuffered "
	SELECT
		json->customer->id AS id,
		json->customer->name AS name
	FROM json
	TO sql
" |
psql -U an_user_name -h a.host.com a_database_name
```

### Monitoring statistics in Kafka

Read data from a kafka topic, continuously calculating statistics.

```sh
kafkacat -b the.broker.com -t the.topic |
spyql --unbuffered "
	SELECT PARTIALS
        count_agg(*) AS running_count,
		sum_agg(value) AS running_sum,
		min_agg(value) AS min_so_far, 
        value AS current_value
	FROM json
	TO csv
" 
```


### Sub-queries (piping)

A special file format (spy) is used to efficiently pipe data between queries.

```sh
cat a_file.json |
spyql "
	SELECT ' '.join([json->first_name, json->middle_name, json->last_name]) AS full_name
	FROM json
	TO spy" |
spyql "SELECT full_name, full_name.upper() FROM spy"
```

### Queries over APIs

```sh
curl https://reqres.in/api/users?page=2 |
spyql "
	SELECT
		json->data->email AS email,
		'Dear {}, thank you for being a great customer!'.format(json->data->first_name) AS msg
	FROM json
	EXPLODE json->data
	TO json
"
```

### Plotting to the terminal

```sh
spyql "
    SELECT col1
    FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
    TO plot
"
```

### Plotting with gnuplot

To the terminal:
```sh
spyql "
    SELECT col1
    FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
    TO csv
" |
sed 1d |
feedgnuplot --terminal 'dumb 80,30' --exit --lines
```

To GUI:

```sh
spyql "
    SELECT col1
    FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
    TO csv
" |
sed 1d |
feedgnuplot --lines --points --exit
```

-----

_This package was created with [Cookiecutter](https://github.com/audreyr/cookiecutter) and the `audreyr/cookiecutter-pypackage` [project template](https://github.com/audreyr/cookiecutter-pypackage)._
