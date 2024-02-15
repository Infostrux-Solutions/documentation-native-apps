Infostrux Loader
================

# Introduction


This application performs the **load (L)** portion of an **extract load transform (ELT)** data pipeline. It works on raw extracts that conform to [Singer open-source standard](https://github.com/singer-io/getting-started).


The application provides a stored procedure that performs the load. The input is a table or view that contains the raw extracted data. The stored procedure reads the raw extracted data, creates target tables with proper schema and populates them. The stored procedure returns the state needed for the next incremental extraction.


The functionality of the app is similar to that of **Singer target**.


The app works well with extractor applications that output data conforming to singer.io standard, such as **Infostrux extractor apps**, but it can be used standalone.


# LOAD Stored Procedure


The stored procedure `LOAD` is available in the `EXECUTION` schema of the application. Assuming the installed application is called `INFOSTRUX_LOADER`, the stored procedure has a signature as follows:


```SQL
INFOSTRUX_LOADER.EXECUTION.LOAD(
   target_database_name VARCHAR,
   target_schema_name VARCHAR,
   source_table_expression VARCHAR
)
```


***target_database_name*** - the name of the database where the target schema and target tables will be created. If the database does not exist, it will be created and owned by the application. If the database does exist, it is assumed that it has been created by the application previously and is owned by it.


***target_schema_name*** - the name of the schema and target tables will be created. If the schema does not exist, it will be created and owned by the application. If the schema does exist, it is assumed that it has been created by the application previously and is owned by it.


***source_table_expression*** - a table expression that returns the data described below. It can be the name of a table or view or a `TABLE(<UDTF>(<params>))` expression or a select statement.


The application takes the raw extracted data as input. It is expected that the raw extracted data contain messages that conform to [Singer standard](https://github.com/singer-io/getting-started/blob/master/docs/SPEC.md). There are three kinds of messages, all expressed as JSON:


- records
- schema information
- state of the extraction


The application uses the schema messages to build table structures. The tables are then populated with the extracted records. The state message is returned for future use.


The ***source_table_expression*** must expose two columns:


```
INDEX       INTEGER
CONTENT     VARCHAR
```


The `CONTENT` column must contain singer.io messages in JSON string format trailed by a new line character. The `INDEX` column must contain an integer that indicates the order in which the messages were extracted. Here is an example from [singer.io GitHub](https://github.com/singer-io/getting-started/blob/master/docs/SPEC.md):




| INDEX | CONTENT |
| ------|---------|
| 1 | {"type": "SCHEMA", "stream": "users", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}, "name": {"type": "string"}}}} |
| 2 | {"type": "RECORD", "stream": "users", "record": {"id": 1, "name": "Chris"}}|
| 3 | {"type": "RECORD", "stream": "users", "record": {"id": 2, "name": "Mike"}} |
| 4 | {"type": "SCHEMA", "stream": "locations", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}} |
| 5 | {"type": "RECORD", "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}  |
| 6 | {"type": "STATE", "value": {"users": 2, "locations": 1}}  |




The extracted data can contain multiple streams. The extract above contains "users" and "locations".




**returns** - the last state from the source. The result can be passed back to some extractors to run incremental loads.






# Configuration and Required Privileges


Consumer data is kept private and secure.


The application needs to be granted an account-level privilege to create databases:


```
GRANT CREATE DATABASE ON ACCOUNT TO APPLICATION INFOSTRUX_LOADER;
```


The application also needs to be granted privileges to execute the `source_table_expression`. The example below shows how to create a sample table and how to grant the privileges:


```SQL


-- create a sample database, schema and temporary table
CREATE DATABASE IF NOT EXISTS EXTRACT_EXAMPLE;
CREATE SCHEMA IF NOT EXISTS EXTRACT_EXAMPLE.OPERATIONS;
CREATE OR REPLACE TEMPORARY TABLE EXTRACT_EXAMPLE.OPERATIONS.DIRECTORY (
   INDEX INT,
   CONTENT VARCHAR
);


INSERT INTO EXTRACT_EXAMPLE.OPERATIONS.DIRECTORY (INDEX, CONTENT)
VALUES
( 1, '{"type": "SCHEMA", "stream": "users", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}, "name": {"type": "string"}}}}
'),
( 2, '{"type": "RECORD", "stream": "users", "record": {"id": 1, "name": "Chris"}}
'),
( 3, '{"type": "RECORD", "stream": "users", "record": {"id": 2, "name": "Mike"}}
'),
( 4, '{"type": "SCHEMA", "stream": "locations", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}}
'),
( 5, '{"type": "RECORD", "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}
'),
( 5, '{"type": "STATE", "value": {"users": 2, "locations": 1}}
');


-- verify that the sample data is properly created
SELECT *, TRY_PARSE_JSON(CONTENT) FROM  EXTRACT_EXAMPLE.OPERATIONS.DIRECTORY;


-- grant the priveleges
GRANT USAGE ON DATABASE EXTRACT_EXAMPLE TO APPLICATION INFOSTRUX_LOADER;
GRANT USAGE ON SCHEMA EXTRACT_EXAMPLE.OPERATIONS TO APPLICATION INFOSTRUX_LOADER;
GRANT SELECT ON TABLE EXTRACT_EXAMPLE.OPERATIONS.DIRECTORY TO APPLICATION INFOSTRUX_LOADER;
```


Please note the new lines in the string literals. These are necessary as the application allows multiple table records per message. The new lines delimit the messages.


# Example Usage


Assuming that the application is installed, sample data shown above was created and the privileges granted, the application can be invoked by simply calling the stored procedure:


```SQL
CALL INFOSTRUX_LOADER.EXECUTION.LOAD('LOAD_TEST', 'OPERATIONS', 'EXTRACT_EXAMPLE.OPERATIONS.DIRECTORY');


-- verify the load
SELECT * FROM LOAD_TEST.OPERATIONS.USERS;
SELECT * FROM LOAD_TEST.OPERATIONS.LOCATIONS;
```


# Notes

The application can be uninstalled, and all databases owned by it dropped by running:

```SQL
DROP APPLICATION INFOSTRUX_LOADER CASCADE;
```

In future releases, we may provide more control over databases and schemas under the application's management.

The application writes INFO-level logs to the event table associated with the account, if present.

