# Introduction

This application performs the **load (L)** portion of an **extract load transform (ELT)** data pipeline. It works on raw extracts that conform to [Singer open-source standard](https://github.com/singer-io/getting-started).

The application provides a stored procedure that performs the load. The input is a table or view that contains the raw extracted data. The stored procedure reads the raw extracted data, creates target tables with proper schema and populates them. The stored procedure returns the state needed for the next incremental extraction.

The functionality of the app is similar to that of **Singer Target**.

The app works well with extractor applications that output data conforming to singer.io standard, such as **Infostrux extractor apps**, but it can be used standalone.

# LOAD Stored Procedure

The stored procedure `LOAD` is available in the `EXECUTION` schema of the application. The signature of the  stored procedure is:

```SQL
LOAD(
  target_database_name VARCHAR,
  target_schema_name VARCHAR,
  source_table_expression VARCHAR
)
```

***target_database_name*** - the name of the database where the target schema and target tables will be created. If the database does not exist, it will be created and owned by the application. If the database does exist, it is assumed that it has been created by the application previously and is owned by it.

***target_schema_name*** - the name of the schema and target tables will be created. If the schema does not exist, it will be created and owned by the application. If the schema does exist, it is assumed that it has been created by the application previously and is owned by it.

***source_table_expression*** - a table expression that returns the data described below. It can be the name of a table or view or a `TABLE(<UDTF>(<params>))` expression or a select statement.

The application takes the raw extracted data as input. The raw extracted data is expected to contain messages that conform to [Singer standard](https://github.com/singer-io/getting-started/blob/master/docs/SPEC.md). There are three kinds of messages, all expressed as JSON:

- records
- schema information
- state of the extraction

The application uses the schema messages to build table structures. The tables are then populated with the extracted records. The state message is returned for future use.

The ***source_table_expression*** must expose two columns:

```
INDEX       INTEGER
CONTENT     VARCHAR
```

The `CONTENT` column must contain singer.io messages in JSON string format trailed by a new line character. The `INDEX` column must contain an integer that indicates the order in which the messages were extracted. Here is an example similar to [singer.io GitHub](https://github.com/singer-io/getting-started/blob/master/docs/SPEC.md):

| INDEX | CONTENT |
| ------|---------|
| 1 | {"type": "SCHEMA", "stream": "person", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}, "name": {"type": "string"}}}} |
| 2 | {"type": "RECORD", "stream": "person", "record": {"id": 1, "name": "Chris"}}|
| 3 | {"type": "RECORD", "stream": "person", "record": {"id": 2, "name": "Mike"}} |
| 4 | {"type": "SCHEMA", "stream": "location", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}} |
| 5 | {"type": "RECORD", "stream": "location", "record": {"id": 1, "name": "Philadelphia"}}  |
| 6 | {"type": "STATE", "value": {"person": 2, "location": 1}}  |

The extracted data can contain multiple streams. The extract above contains streams "person" and "location".

**returns** - the last state from the source. The result can be passed back to some extractors to run incremental loads.

# Configuration and Required Privileges

Consumer data is kept private and secure.

The application needs to be granted an account-level privilege to create databases:

```
set application_name = '<application name>';

-- grant the required privileges
grant create database on account to application identifier($application_name);
```

Please adjust the script by replacing the string `<application name>` so that the session variable `application_name` refers to the application as it was installed in your account. All scripts below assume that the session variable is available.

The application must be granted enough privileges to execute the `source_table_expression`. For example, if the `source_table_expression` references a table, the application must be granted `USAGE` privilege on the database and schema that contains the table and `SELECT` privilege on the table itself. See the examples below.

# Example

The native app provides table `sample_extract` in `execution` schema that holds the sample raw extracted data shown above. Run the following script to use it:

```SQL
use database identifier($application_name);
use schema execution;

-- Verify that the sample data is available.
-- Note the trailing new lines in the CONTENT column.
-- These are necessary as the application allows multiple table records per message.
-- The new lines delimit the messages.
select *, try_parse_json(content)
from sample_extract;

-- Invoke the loader on sample data.
call load(
   'load_test',
   'app_example',
   'sample_extract');

-- Verify that the sample extract was correctly loaded.

-- two records with id and name
select * from load_test.app_example.person;

-- a single record with id and no other columns
select * from load_test.app_example.location;

```

# Notes

The application can be uninstalled, and all databases owned by it dropped by running:

```SQL
drop application identifier($application_name) cascade;
```

In future releases, we may provide more control over databases and schemas under the application's management.

The application writes INFO-level logs to the event table associated with the account, if present.

