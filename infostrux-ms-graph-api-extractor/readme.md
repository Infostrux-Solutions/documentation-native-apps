Infostrux MS Graph API Extractor
================================

# Introduction

The app extracts data from Microsoft Entra (formerly known as Azure AD) via Microsoft Graph API. It extracts users, groups, group membership, SKUs and sign-in log entries.

The app provides a user-defined table function (UDTF) that returns raw extracted data conforming to the open-source Singer protocol. Infostrux Loader app or other loading mechanisms can be used to load the data into structured tables.

# EXTRACT() UDTF

The application exposes `EXTRACT` UDTF in the `EXECUTION` schema. The UDTF reruns the extracted data. Assuming that the installed application is called `INFOSTRUX_MS_GRAPH_API_EXTRACTOR`, the function signature is:

```SQL
INFOSTRUX_MS_GRAPH_API_EXTRACTOR.EXECUTION.EXTRACT()
```

Once configured, the function can be used in select statements. For example:

```SQL
SELECT * FROM (TABLE(INFOSTRUX_MS_GRAPH_API_EXTRACTOR.EXECUTION.EXTRACT()));
```
The returned table contains the following columns.

- `CONTENT    VARCHAR` - text containing partial or full Singer record in Singer format. The records in Singer format are separated by a new line character, which will be present in the `CONTENT` field. It is usually the last character.

- `TIMESTAMP TIMESTAMP_NTZ` - the timestamp in UTC when the individual record was emitted

- `INDEX NUMBER` - an increasing sequence of numbers indicating the order of records in Singer extract

# INIT_EXTRACT() Stored Procedure

The application exposes the `INIT_EXTRACT` Stored Procedure in the `INITIALIZATION` schema. The stored procedure allows the EXTRACT() UDTF to use supplied integration and config secrets. Assuming that the installed application is called `INFOSTRUX_MS_GRAPH_API_EXTRACTOR`, the stored procedure signature is:


```SQL
INFOSTRUX_MS_GRAPH_API_EXTRACTOR.INITIALIZATION.INIT_EXTRACT(
 integration_name VARCHAR,
 config_secret_full_name VARCHAR
)
```

The stored procedure needs to be called only once. Once the initialization is complete, EXTRACT() UDTF can be called multiple times, and it will perform data extraction on every call.

See the section below for an example.


# Configuration and Initialization

The function needs to be configured by:
- granting the use of an external access integration that allows access to MS Graph API end-points
- providing connection credentials as secrets associated with the integration

Here is a script for that purpose. Please edit the settings as appropriate. Note that there is one place in the script marked with "!!!" that needs to be adjusted if settings are modified:

```SQL
-- the role that will be executing the script.
-- the role needs grants to create integrations and access to the config db
SET role_name = 'ACCOUNTADMIN';
-- warehouse for testing the functionality
SET warehouse_name = 'APP_TEST_WAREHOUSE';             

-- database that will hold the configuration secret
SET database_name ='INFOSTRUX_MS_GRAPH_API_EXTRACTOR_CONFIG'; 
-- schema that will hold the configuration secret
SET schema_name = 'CONFIG';                          

-- integration, network rule, and config secret for accessing MS Graph API
SET integration_name = 'MS_GRAPH_API_INTEGRATION';
SET network_rule_name = 'MS_GRAPH_API_NETWORK_RULE';
SET config_secret_name = 'MS_GRAPH_API_CONFIG_SECRET';
SET config_secret_full_name = $database_name || '.' || $schema_name || '.' || $config_secret_name;

-- the actual config secret that holds the credentials
SET config_secret = '{
   "tenant" : "...",       
   "client_id" : "...",
   "client_secret" : "..."
}';

-- the name of the app as installed from the marketplace
SET app_name = 'INFOSTRUX_MS_GRAPH_API_EXTRACTOR';  
SET init_proc_name = $app_name || '.INITIALIZATION.INIT_EXTRACT';
SET extract_function_name = $app_name || '.EXECUTION.EXTRACT';


-----
--- execution

-- role and warehouse
USE ROLE identifier($role_name);

CREATE WAREHOUSE IF NOT EXISTS identifier($warehouse_name)
WITH
   WAREHOUSE_SIZE = XSMALL
   INITIALLY_SUSPENDED=TRUE
   AUTO_SUSPEND = 60
   AUTO_RESUME = TRUE ;

USE WAREHOUSE identifier($warehouse_name);

-- config database
CREATE DATABASE IF NOT EXISTS identifier($database_name);
USE DATABASE identifier($database_name);

CREATE SCHEMA IF NOT EXISTS identifier($schema_name);
USE SCHEMA  identifier($schema_name);

-- network rule and external network access integration
CREATE OR REPLACE NETWORK RULE identifier($network_rule_name)
 MODE = EGRESS
 TYPE = HOST_PORT
 VALUE_LIST = ('graph.microsoft.com', 'login.microsoftonline.com');

CREATE OR REPLACE SECRET identifier($config_secret_full_name)
  TYPE = GENERIC_STRING
  SECRET_STRING = $config_secret;

DROP INTEGRATION identifier($integration_name);
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION identifier($integration_name)
 ALLOWED_NETWORK_RULES = ( $network_rule_name )
 -- !!! HAS TO BE SPELLED OUT AND MODIFIED IF NEEDED - identifier() syntax does not work here
 ALLOWED_AUTHENTICATION_SECRETS = (ms_graph_api_config_secret)  
 ENABLED = true;

-- grants to the app
GRANT USAGE ON DATABASE identifier($database_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON SCHEMA identifier($schema_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON SECRET identifier($config_secret_full_name) TO APPLICATION identifier($app_name);
GRANT READ ON SECRET identifier($config_secret_full_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON INTEGRATION identifier($integration_name) TO APPLICATION identifier($app_name);

-- call to the initialization stored proc that sets up the EXTRACT() UDTF
CALL identifier($init_proc_name)(
   $integration_name,
   $config_secret_full_name);

-- call to the EXTRACT() UDTF
-- returns a list of Singer messages as extracted from the API
SELECT * FROM TABLE(identifier($extract_function_name)());
```

## How to Obtain Microsoft Entra (Formerly Azure AD) Credentials

To extract SignIn logs, your Microsoft Entra (formerly Azure AD) needs to have an Azure AD Premium P1 or P2 license. To verify the licensing requirements, sign in to [Entra admin centre](https://entra.microsoft.com/#view/Microsoft_AAD_IAM/TenantOverview.ReactView) and navigate to Identity / Overview section. It will show the license under the Overview tab in the Basic Information section of the page.

Follow the instructions in [Register an application with the Microsoft identity platform](https://learn.microsoft.com/en-us/graph/auth-register-app-v2) to register a new application with the identity platform. You don't need to [Configure Platform Settings](https://learn.microsoft.com/en-us/graph/auth-register-app-v2#configure-platform-settings) nor to add a redirect url. You do need to [Add credentials](https://learn.microsoft.com/en-us/graph/auth-register-app-v2#add-credentials). Use [Option 2: Add a client secret](https://learn.microsoft.com/en-us/graph/auth-register-app-v2#option-2-add-a-client-secret).

Copy the values to your setup script

```JSON
{
   "tenant": "< from overview page >",       
   "client_id": "< from overview page >",
   "client_secret": "< the client secret value (not the id)"
}
```

Follow the section [Configure permissions for Microsoft Graph](https://learn.microsoft.com/en-us/graph/auth-v2-service?tabs=http#2-configure-permissions-for-microsoft-graph) to add application permissions (API permissions) for the entities you want to read. They are going to be under Microsoft Graph API. Review the permissions on your app page and grant admin consent on that page. You don’t need to follow any other sections from that page.

The necessary permissions are

- AuditLog.Read.All
- Directory.Read.All
- Group.Read.All
- GroupMember.Read.All
- IdentityProvider.Read.All
- User.Read
- User.Read.All

# Notes
As shown in the configuration script above, the application needs usage permissions to the application integration and usage and read permissions on the credential secrets as well as usage permissions to the database and schema that store the secrets:

```SQL
GRANT USAGE ON DATABASE identifier($database_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON SCHEMA identifier($schema_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON SECRET identifier($config_secret_full_name) TO APPLICATION identifier($app_name);
GRANT READ ON SECRET identifier($config_secret_full_name) TO APPLICATION identifier($app_name);
GRANT USAGE ON INTEGRATION identifier($integration_name) TO APPLICATION identifier($app_name);
```

Note that once the secrets are granted and the application is initialized, anybody who has usage permissions on the app will be able to extract and read the data from your identity platform without requiring permissions for integration and/or secrets themselves. The grantors will not be able to read the secrets or use the integration themselves unless they are granted to them explicitly elsewhere.
