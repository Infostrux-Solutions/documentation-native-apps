Infostrux Modern Software Asset Management
================================

# Introduction

This native app was designed to help you save money on third-party SaaS and on-prem license costs.

According to the 2023 State of ITAM Report, nearly one-third of SaaS licenses purchased by organizations go underutilized or wasted. Deprovisioning licenses for underutilized assets such as productivity suites, charting and diagramming tools, line of business or sales support, development tools and others can lead to very substantial savings.

The native app uses SSO sign-in logs to analyze your organization's third-party application usage. Based on the analysis, the native app provides a third-party application usage overview showing how many users signed in to each third-party application in the last 30, 60, 90, 150+ days. For a selected third-party application, the native app lists the individual users by last sign-in date, indicating which users can be considered for de-provisioning and potential license cost savings.

# Setup

Once the native app is installed, it provides a streamlit in the native app’s `CORE` schema. The app is pre-configured with sample data. The preview of the application is fully functional and does not require any further setup.

To setup the application to use your organization’s SSO sign-in logs, invoke the native app’s `SETUP()` stored procedure in the `CONFIGURATION` schema:

```SQL

call <application_name>.configuration.setup(
   '<database_name>',
   '<schema_name>');
```

Where
`<application_name>` is the name of this native app installed in your account
`<database_name>` is the name of the database that holds your sign-in logs
`<schema_name>` is the name of the schema that holds your sign-in logs

The schema must contain `USERS` and `SIGNINS` tables with the following columns:

``` SQL
USERS

name                type
---------------------------------
ID                  VARCHAR
DISPLAYNAME         VARCHAR
GIVENNAME           VARCHAR
SURNAME             VARCHAR
MAIL                VARCHAR
```

``` SQL
SIGNINS

name            type
---------------------------------
ID                  VARCHAR
USERID              VARCHAR
APPID               VARCHAR
APPDISPLAYNAME      VARCHAR
ISINTERACTIVE       NUMBER(1,0)
CREATEDDATETIME     DATETIME
```

The source data is compatible with MS Graph API extracts from Microsoft Entra (formerly Azure AD) but can be populated from other SSO sources.

Infostrux provides a free-to-use extract native app "Infostrux MS Graph API Extractor”, and a free-to-use loader native app "Infostrux Loader," that can be used to extract and load the data, but their use is optional.

Before the `SETUP()` stored proc is called, the application must be granted `USAGE` privilege on the database and schema and `SELECT` privilege on the tables. Here's a sample script:

```SQL
----------------
-- variables

set source_database = '<database that holds the loaded data>';
set source_schema = '<schema that contains USERS and SIGNINS tables or views>';

set app_name = '<application name>';

set source_qualified_schema = $source_database || '.' || $source_schema;
set source_users = $source_qualified_schema || '.users';
set source_signins = $source_qualified_schema || '.signins';

---------------
-- grants

grant usage on database identifier($source_database) to application identifier($app_name);
grant usage on schema identifier($source_qualified_schema) to application identifier($app_name);
grant select on table identifier($source_users) to application identifier($app_name);
grant select on table identifier($source_signins)  to application identifier($app_name);

----------------
-- setup

use database identifier($app_name);

call configuration.setup(
    $source_database,
    $source_schema);

```

# Notes

The application can be uninstalled, and all objects owned by it dropped by running:

```SQL
drop application <application_name> cascade;
```

The application may write INFO-level logs to the event table associated with your account if present.

