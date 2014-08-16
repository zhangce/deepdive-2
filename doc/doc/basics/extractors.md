---
layout: default
---

# Writing extractors

Extractors are a powerful functionality provided by DeepDive to streamline
[feature extraction](overview.html#extractors). This document presents the
different types of extractors supported by DeepDive. Please refer to the
[Configuration reference](configuration.html#extractors) for a more structured
presentation of each definition directive.

## Styles of extractors

DeepDive supports two classes of extractors: *row-wise* and *procedural*. Each
class contains different extractor *styles*:

- Row-wise extractors:
  - [`json_extractor`](#json_extractor): highly flexible and compatible with
	previous systems, but with limited performance 
  - [`tsv_extractor`](#tsv_extractor): moderate flexibility and performance
  - [`plpy_extractor`](#plpy_extractor): parallel database-built-in extractors
	with restricted flexibility

- Procedural extractors:
  - [`sql_extractor`](#sql_extractor): a SQL command
  - [`cmd_extractor`](#cmd_extractor): a shell command

Row-wise extractors perform a user-defined function (UDF) on each tuple in the
results of a input query against the database. One may think of these extractors
as functions mapping  one input tuple to one or more output tuples, similar to
a `map` or `flatMap` function in functional programming languages. Procedural
extractors are arbitrary SQL or shell commands.

Extractors are specified in the `extraction.extractors` section of the
application configuration file. Each extractor is defined in a section that
starts with the name of the extractor and containing the specifications for the
instructor.

```bash
deepdive {
  extraction.extractors {
	  
    anExtractor {
      # ...
    }

    anotherExtractor {
      # ...
	}

	# More Extractors ...
  }
}
```

The style of an extractor is specified using the `style` keyword in each
extractor definition in the application configuration file. In the following
example, `wordsExtractor` is a `tsv_extractor`:

```bash
deepdive {
  extraction.extractors {

    wordsExtractor {
      style: "tsv_extractor"
      # ...
    }
    # More Extractors...
  }
}
```

If `style` is not specified, the system assumes the extractor has style `json_extractor`.

### <a name="json_extractor" href="#"></a> json_extractor (default)

A `json_extractor` takes each tuple in the output of an `input` query (for
example, a SQL statement), and produces new tuples as output. These tuples are
written to an `output_relation`. The transformation function is defined by the
value of the `udf` keyword, which can be an arbitrary executable. 

<!-- TODO (MR) can it be a shell command ? -->

The following is an example of an extractor definition:

```bash
wordsExtractor {
  style           : "json_extractor"
  input           : """SELECT * FROM titles"""
  output_relation : "words"
  udf             : "words.py"
}
```

#### Input to a json_extractor

Currently DeepDive supports two types of extractor inputs for `json_extractor`:

**1. Executing a database query**

A SQL statement for PostgresSQL, as in the example above.

**2. Reading from a CSV or TSV File**

Reading a file is useful for loading initial data. To specify which file to read
and its format, the `input` directive of the extractor definition should look
like the following:

<!-- TODO (MR) Check that these are correct -->
```bash
input: CSV('path/to/file.csv')
input: TSV('path/to/file.tsv')
```
#### Writing the UDF for a json_extractor

When a `json_extractor` is executed and if the `input` directive is a SQL query,
DeepDive streams *[JSON](http://json.org/) objects* from the specified `input`
to the *standard input* of the extractor UDF, one tuple per line. Such an object
may look as follows:

    { titles.title_id: 5, titles.title: "I am a title" }

Columns names are always prefixed with the name of the table. For example, if
your query includes a `name` column from the `people` table, then the
corresponding JSON key would be called `people.name`. This also applies to
aliases. For example, `SELECT people.name AS text` would result in a JSON key
called `people.text`. Aggregates are prepended with a dot and do not include the
relation name.  For example `SELECT COUNT(*) AS num FROM people GROUP BY
people.name` would result in a JSON key called `.num`.


If instead the `input` directive specifies reading from a CSV or TSV file, each
line streamed to the standard input of the UDF is an *array*, as in:

    ["1", "true", "Hello World", ""]

The extractor UDF must output JSON objects to the *standard output*, one per
line, independently of the format of the input. All output objects must have the
same fields, so it may necessary to set the values for some fields to `null`.
The following is an example of extractor output:

    { title_id: 5, word: "I" } 
    { title_id: 5, word: "am" } 
    { title_id: 5, word: "a" } 
    { title_id: 5, word: "title" } 
	{ title_id: 6, word: null }

When emitting tuples from the extractor, only use the column name of the
`output` relation, without the relation name. In other words, do not use JSON
keys like `people.name`, but keys with names like `name`.

You can debug the extractor UDF by writing to `stderr` instead of `stdout`.
Anything written to `stderr` appears in the DeepDive log file.

The following is an example of a`json_extractor` UDF written in Python:

```python
#! /usr/bin/env python

import fileinput
import json

# For each input row
for line in fileinput.input():
  # Load the JSON object
  row = json.loads(line)
  if row["title"] is not None:
    # Split the sentence by space
    for word in set(row["title"].split(" ")):
      # Output the word
      print json.dumps({
        "title_id": int(row["title_id"]), 
        "word": word
      })
```

### <a name="tsv_extractor" href="#"></a> tsv_extractor

A `tsv_extractor` is very similar to the default `json_extractor`, but its
performance is optimized:
[TSV](https://en.wikipedia.org/wiki/Tab-separated_values) is used instead of
JSON to achieve higher processing speed. A `tsv_extractor` takes data defined by an
`input` query and produces new tuples as output. These tuples are written to the
`output_relation`. The function for this transformation can be any executable
defined in `udf`:

    wordsExtractor {
      style           : "tsv_extractor"
      input           : """SELECT * FROM titles"""
      output_relation : "words"
      udf             : "words.py"
    }

Extractors with this style *must* have a SQL query as `input` (not a TSV or CSV
file).

When DeepDive executes an extractor with this style, the following happens:

1. The results of the input query are unloaded into multiple TSV files.
<!-- TODO (MR) is there a way to specify how many TSV files ? -->
2. Multiple instances of the extractor UDF are executed in parallel, with the
TSV files piped into the standard input of these instances.
3. The outputs (to standard output) of the extractor UDF instances (also in
TSV format) are loaded into the database with a COPY command.

#### Input to a tsv_extractor

The `input` to a `tsv_extractor` must be a database query. For example

```sql
SELECT title_id, title FROM title;
```

The order of the columns in the query imposes the order of the columns in the
TSV file given in input to the UDF. For the query example above, each line will
contain first the `title_id` and then `title`.

#### Arrays in input or output

If the output of the input query contains arrays, it will be hard to parse the
reuslting `TSV` files will be hard to parse. In this case it is recommended to
use the `array_to_string` function to process the array in the input query, and
then parse the string in UDF.

For example, for an input query like this:

```sql
SELECT words_id, array_to_string(words, '$$$') FROM words_table;
```

You can parse each line with following:

```python
words_id, words_str = line.split('\t')
words = words_str.split('$$$')
for word in words:
  # process each word...
```

Note that if the output of the UDF contains arrays, the database system may have
an hard time parsing it. You should either make sure that the value can
be parsed by
the psql-COPY command, or try other types of extractors.

#### Writing UDF a `tsv_extractor`

When an `tsv_extractor` is executed, DeepDive streams plaintext lines with
fields separated by `\t` from the input query to the UDF standard input, one
tuple per line. Such a line may look as follows:

    "5\tI am a title"

The extractor should output TSV lines in the same fashion. All
output tuples must have the same fields. Empty fields can set to `"\N"`, which
is be parsed as `null` by the database COPY command:

    "5\tI"
    "5\tam"
    "5\ta"
    "5\ttitle"
    "6\t\N"  # example of returning a NULL value

You can debug the extractor UDF by writing to `stderr` instead of `stdout`.
Anything written to `stderr` appears in the DeepDive log file.

The following is an example of a `tsv_extractor` UDF written in Python:

```python
#! /usr/bin/env python

import fileinput

# For each input row
for line in fileinput.input():
  title_id, title = line.split('\t')
  if title is not None:
    # Split the sentence by space
    for word in set(row["title"].split(" ")):
      # Output the word
      print title_id + '\t' + word
```

### <a name="plpy_extractor" href="#"></a> plpy_extractor

A `plpy_extractor` is a high-performance type of extractors for PostgreSQL /
Greenplum databases. It avoids additional I/O by executing the extractor
**inside** the database system in parallel. It translates an UDF written by the
user in Python into a
[PL/python](http://www.postgresql.org/docs/9.3/static/plpython.html) program
accepted by PostgreSQL.

To use a `plpy_extractor`, make sure PL/Python is enabled on your database
server.

Similar as `tsv_extractor`, the UDF takes data defined by an `input`
SQL statement, and produces new tuples as output. These tuples are written to
the `output_relation`. The UDF for this transformation, defined by the `udf`
key, **must** be a Python script with a specific format (more on
this later). The following is an example of a `plpy_extractor` definition:

```bash
# An extractor to get trigrams of words from sentences to word_3gram table
ngramExtractor {
  style           : "plpy_extractor"
  input           : """SELECT sentence_id, words, 3 as gram_len FROM sentences"""
  output_relation : "word_3gram"
  udf             : "ext_word_ngram.py"
}
```

#### Writing the UDF for a `plpy_extractor`

Since it will be translated into PL/Python by translator in
DeepDive, the UDF script of a `plpy_extractor` **must** follow a specific
structure:

- It must contain only two functions: `init` and `run`. Anything outside the
  `init` and `run` functions will be ignored by the translator.

- In the `init` function, you should import libraries, and specify the input
  variables and return types.

  1. You can **import libraries** used by the UDF by calling the function
  `ddext.import_lib`. The signature of the function is: `def import_lib(libname,
  from_package=None, as_name=None)` and corresponds, in standard Python syntax,
  to `from from_package import libname as as_name`. A sample usage is the
  following:
      - `ddext.import_lib(X, Y, Z)`: from Z import X as Y
      - `ddext.import_lib(X, Y)`: from Y import X
      - `ddext.import_lib(X, as_name=Z)`: import X as Z
      - `ddext.import_lib(X)`: import X

  2. The **input variables** to the UDF must be explicitly specified with
  `ddext.input`. You must specify both variable **names** and **types**. The
  function is defined as: `def input(name, datatype)`. A sample usage is:
	  - `ddext.input('sentence_id', 'bigint')` specifies an input to UDF with
		name "sentence_id" and type "bigint".
  Caveats:
	  - **Input variable names must be the same as in the SQL input query
		(aliased), and the types must match.**
	  - Names should be coherent to the argument list of `run` function (see
		below).
	  - Types are PostgreSQL types, e.g., `int`, `bigint`, `text`, `float`,
		`int[]`, `bigint[]`, `text[]`, `float[]`, etc.
        
  3. The **return types and names** must be explicitly specified with
  `ddext.returns`. This function is defined as: `def returns(name, datatype)`. A
  sample usage is:
		  - `ddext.returns('sentence_id', 'bigint')` specifies an output from
			UDF with name "sentence_id" and type "bigint".
  Caveats:
          - Types are PostgreSQL types as above.
		  - **Names and types** of return variables must **exactly match**
			some columns of the extractors `output_relation`. If the output
			relation contains more columns, the values of the tuples in the
			unspecified columns will be NULL.

- The `run` function is your extractor function that takes one row in
  the `input` SQL query`, and returns a list of tuples. Use Python as you
  normally would, except for the following caveats:
		- `print` is NOT supported. If you want to print to the DeepDive log
		  file, use `plpy.info('SOME TEXT')` or `plpy.debug('SOME TEXT')`.
        - You can not **reassign input variables** in the `run` function! 
            - e.g., "input_var = x" is invalid and will cause error!
        - Libraries imported in the `init` function can be used in `run`. 
			- e.g., if you have `ddext.import_lib('re')` in `init`, you can call
			  the function `re.sub` in `run`.
  The function `run` can either return a list of tuples:
            ```python
            return [(sentence_id, gram, ngram[gram]) for gram in ngram]
            ```
  or `yield` a tuple multiple times. Each tuple it yields will be inserted into
  the database, just like each printed JSON object in a json_extractor. Each
  yielded/returned tuple can be either:
        - an ordered list / tuple according to the order of `ddext.return` specification: 

            ```python
            yield sentence_id, gram, ngram[gram]
            ```

        - a Python `dict`:

            ```python
            yield {
                'sentence_id': sentence_id, 
                'ngram': ngram, 
                'count':ngram[gram]
              }```
  If you want to use functions other than `init` and `run`, you **must**
	define the functions inside `run`as nested functions. In the following
	example, the `get_ngram` function is nested inside `run`:

        ```python
        def run(sentence_id, words):
          ngram = {}

          # Count Ngrams of words; N as input
          # words / ngram is accessible in the function
          def get_ngram(N):
            for i in range(len(words) - N):
              gram = ' '.join(words[i : i + N])
              
              if gram not in ngram: 
                ngram[gram] = 0
              ngram[gram] += 1

          for n in range(1, 5):
            get_ngram(n)

          return [(sentence_id, key, ngram[key]) for key in ngram]
        ```

You can debug `plpy_extractors`using *plpy.info* or *plpy.debug*
instead of *print*. The output will appear in the DeepDive log file.

As an example, the following is the `ext_word_ngram.py` script used by the
extractor `ngramExtractor` defined above:

```python
#! /usr/bin/env python

import ddext

def init():
  ddext.input('sentence_id', 'bigint')
  ddext.input('words', 'text[]')
  ddext.input('gram_len', 'int')

  ddext.returns('sentence_id', 'bigint')
  ddext.returns('ngram', 'text')
  ddext.returns('count', 'int')

def run(sentence_id, words, gram_len):
  # Count Ngrams of words in the sentence
  ngram = {}
  for i in range(len(words) - gram_len):
    gram = ' '.join(words[i : i + gram_len])
    if gram not in ngram: 
      ngram[gram] = 0
    ngram[gram] += 1
    
  for gram in ngram:
    # Yield an ordered tuple/list:
    yield (sentence_id, gram, ngram[gram])

    # Or yield a (unordered) dict:
    # yield {
    #     'sentence_id': sentence_id, 
    #     'ngram': ngram, 
    #     'count':ngram[gram]
    #   }

  # # Or return a list at once:
  # return [[sentence_id, gram, ngram[ngram]] for gram in ngram] 
```

### <a id="sql_extractor" href="#"></a> sql_extractor

The `sql_extractor` style is a meant to simplify the life of the user.
Extractors with this style only update the data in database (without returnining
anything). The following is an example of a `sql_extractor` definition:

```bash
wordsExtractor {
  style : "sql_extractor"
  sql   : """INSERT INTO titles VALUES (1, 'Moby Dick')"""
}
```

The `sql` field contains the SQL query to execute. It is the user's
responsibility to run `ANALYZE table_name` after updating the table for SQL
optimization.

### <a id="cmd_extractor" href="#"></a> cmd_extractor

The `cmd_extractor` style allows the user to run a shell command. The following
is an example of a `cmd_extractor` definition:

```bash
wordsExtractor {
  style : "cmd_extractor"
  cmd   : """python words.py"""
}
```

The `cmd` field contains the shell command to execute.

## Additional extractor properties

The following are additional properties of extractors that you can specify in
the application configuration file.

### <a name="dependencies" href="#"></a> Dependencies

You can specify dependencies for an extractor. Extractors will be executed
in order of their dependencies. If the dependencies of several extractors are
satisfied at the same time, these may be executed in parallel, or in any order.

```bash
wordsExtractor {
  dependencies: ["anotherExtractorName"]
}
```

If an extractor specified in dependencies does not exist or is not in the
active [pipeline](running.html#pipelines), that extractor will be ignored. 

### <a name="parallelism" href="#"></a> Parallelism
You can execute independent extractors in parallel:

<!-- TODO Shouldn't common configs be at deepdive.extraction level, e.g.,
deepdive.extraction.parallelism?  Otherwise, config keys may collide with
user-defined extractor names. -->

<!-- TODO (MR) -->

```bash
deepdive {
  extraction.extractors {
    parallelism: 5

    # Extractors...
  }
}
```

### <a name="beforeandafter" href="#"></a> Before and After scripts

Sometimes it is useful to execute a command before an extractor starts or after
an extractor finishes. You can specify arbitrary commands to be executed as
follows:

```bash  
wordsExtractor {
  before : """echo Hello World"""
  after  : """/path/to/my/script.sh"""
  # ...
}
```

### <a name="jsonparallelism" href="#"></a> `json_extractor` parallelism and input batch size 

*Note*: This section applies **only** to extractors with style `json_extractor`.

To improve the performances, you can specify the number `N` of processes and the
input batch size for each extractor. Your executable script will be run on `N`
threads in parallel and data will be streamed to these processes in a
round-robin fashion. By default each extractor uses 1 process and a batch size
of 1000.

```bash
wordsExtractor {
  # Start 5 processes for this extractor
  parallelism: 5
  # Stream 1000 tuples to each process in a round-robin fashion
  input_batch_size: 1000
}
```

To improve the performances in writing extracted data back to the database you
can optionally specify an `output_batch_size` for each extractor. The output
batch size specifies how many extracted tuples we insert into the database at
once.  For example, if your tuples are very large, a smaller batch size may help
avoid out-of-memory errors. The default value is 10,000.

```bash
wordsExtractor {
  # Insert 5000 tuples at a time into the data store
  output_batch_size: 5000
}
```
