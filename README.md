# SpyQL: SQL with Python in the middle

[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

> **ATTENTION**: SpyQL has not been released yet. Reach me out if you would like to contribute (look at the Issues list)!

## Concept

SpyQL is a query language that combines:

* the simplicity and structure of SQL
* with the power and readability of Python 

```sql
SELECT
    date.fromtimestamp(purchase_ts) AS purchase_date,
    price * quantity AS total 
FROM csv 
WHERE department.upper() == ‘IT' 
TO json 
```

SQL provides the structure of the query, while Python is used to define expressions, bringing along a vast ecosystem of packages.


### SpyQL command-line tool 

With the SpyQL command-line tool you can make SQL-like SELECTs powered by Python on top of text data (e.g. CSV and JSON). Data can come from files but also from data streams, such as as Kafka, or from databases such as PostgreSQL. Basically, data can come from any command that outputs text :-). More, data can be generated by a Python iterator! Take a look at the examples section to see how to query parquet, process API calls, transverse directories of zipped JSONs, among many other things.

SpyQL also allows you to easily convert between text data formats: 

* `FROM`: CSV, JSON, TEXT and Python iterators (YES, you can use a list comprehension as the data source)

* `TO`: CSV, JSON, TEXT, SQL (INSERT statments), pretty terminal printing, and terminal plotting. 

The JSON format is [JSON lines](https://jsonlines.org), where each line has a valid JSON object or array. Piping with [jq](https://stedolan.github.io/jq/) allows SpyQL to handle any JSON input (more on the examples section).

You can leverage command line tools to process other file types like Parquet and XML  (more on the examples section).


## Principles

Right now, the focus is on building a command-line tool that follows these core principles:

* **Simple**: simple to use with a straifghtforward implementation
* **Familiar**: you should feel at home if you are acquainted with SQL and Python
* **Light**: small memory footprint that allows you to process large data that fit into your machine
* **Useful**: it should make your life easier, filling a gap in the eco-system


## Syntax

```sql
SELECT 
    [ * | python_expression [ AS output_column_name ] [, ...] ]    
    [ FROM csv | spy | text | python_expression | json [ EXPLODE path ] ]
    [ WHERE python_expression ]
    [ LIMIT row_count ]
    [ OFFSET num_rows_to_skip ]
    [ TO csv | json | text | spy | sql | pretty | plot ]
```

Comming next: `ORDER BY`, `GROUP BY`, `EXECUTE`


## Notable differences to SQL

In SpyQL:

* there is guarantee that the order of the output rows is the same as in the input 
* the `AS` keyword must precede a column alias definition (it is not optional as in SQL)
* you can always access the nth column by using the default column names `colN` (e.g. `col1` for the first column)
* currently only a small subset of SQL is supported, namely `SELECT` statements without: sub-queries, joins, set operations, etc (check the [Syntax](#syntax) section)
* sub-queries are achieved by piping (see the [Command line examples](#command line examples)
 section)
* expressions are pure Python:

| SQL | SpySQL |
| ------------- | ------------- |
| `x = y` | `x == y` |
| `x LIKE y` | `like(x, y)`  |
| `x BETWEEN a AND b`  |  `a <= x <= b` | 
| `CAST(x AS INTEGER)` | `int(x)` |
| `CASE WHEN x > 0 THEN 1 ELSE -1 END` | `1 if x > 0 else -1` |
| `upper(‘hello')` | `'hello'.upper()` |


## Notable differences to Python

### Additional syntax

We added additional syntax for making querying easier: 

| Python | SpySQL shortcut| Purpose |
| ------ | -------------- | ------- |
| `json[‘hello'][‘planet earth']` | `json->hello->‘planet earth'` | Easy access of elements in  dicts (e.g. JSONs) |


### NULL datatype

Python's `None` generates exceptions when making operations on missing data, breaking query execution (e.g. `None + 1` throws a `TypeError`). To overcome this, we created a `NULL` type that has the same behavior as in SQL (e.g. `NULL + 1` returns `NULL`), allowing for queries to continue processing data.

|Operation | Native Python throws | SpySQL returns | SpySQL warning |
| ------------- | ------------- | ------------- | ------------- |
| `NULL + 1` | `NameError` | `NULL` | |
| `adict[‘inexisting_key']` | `KeyError` | `NULL` | yes | 
| `int(‘')`   |  `ValueError` | `NULL` | yes | 
| `int(‘abc')` | `ValueError` | `NULL` | yes | 

The above dictionary key access only returns `NULL` if the dict is an instance of `NullSafeDict`. SpyQL adds `NullSafeDict`, which extends python's native `dict`. JSONs are automatically loaded as `NullSafeDict`. Unless you are creating dictionaries on the fly you do not need to worry about this.

## Example queries

### Query a CSV (and print a pretty table)

```sql
SELECT a_col_name, ‘positive' if col2 >= 0 else ‘negative' AS sign
FROM csv 
TO pretty
``` 

### Convert CSV to a flat JSON 

```sql
SELECT * FROM csv TO json 
``` 

### Convert from CSV to a hierachical JSON

```sql
SELECT {'client': {'id': col1, ‘name': col2}, ‘price': 120.40} 
FROM csv TO json 
``` 

or

```sql
SELECT {'id': col1, ‘name': col2} AS client, 120.40 AS price 
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
```

or

```sql
SELECT col1     
FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)] 
```

### Python multi column iterator to CSV

TODO Example of Python multi column iterator to CSV


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

[yq](https://kislyuk.github.io/yq/#) converts yaml, xml and toml files to json, allowing to easily query any of these with spyql: 

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

Read data from a kafka topic and write to postgres.

NOTE: sql  options not yet implemented, i.e. output table name and number of records per insert are hardcoded

```sh
kafkacat -b the.broker.com -t the.topic | 
spyql "
	SELECT 
		json->customer->id AS id,
		json->customer->name AS name
	FROM json
	TO sql('customer_table_name', chunk_size=1)
" |
psql -U an_user_name -h a.host.com a_database_name
```

### Sub-queries (piping)

A special file format (spy) is used to efficiently pipe data between queries.

```sh
cat a_file.json |
spyql "
	SELECT ' '.join[json->first_name, json->midle_name, json->last_name] AS full_name
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
		‘Dear {}, thank you for being a great customer!'.format(json->data->first_name) AS msg  
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

```sh
spyql "
    SELECT col1
    FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
    TO csv
" |
sed 1d |
feedgnuplot --terminal ‘dumb 80,30' --exit --lines 
```

```sh
spyql "
    SELECT col1
    FROM [10 * cos(i * ((pi * 4) / 90)) for i in range(80)]
    TO csv
" | 
sed 1d | 
feedgnuplot --lines --points --exit
```
