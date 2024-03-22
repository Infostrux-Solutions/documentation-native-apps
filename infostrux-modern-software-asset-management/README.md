Infostrux Modern Software Asset Management
================================

# Introduction


This native app was designed to help you save money on third-party SaaS and on-prem license costs.


According to the 2023 State of ITAM Report, nearly one-third of SaaS licenses purchased by organizations go underutilized or wasted. Deprovisioning licenses for underutilized assets such as productivity suites, charting and diagramming tools, line of business or sales tools, and development tools can lead to substantial savings.


The native app uses SSO sign-in logs to analyze your organization's third-party application usage. Based on the analysis, the native app provides a third-party application usage overview showing how many users signed in to each third-party application in the last 30, 60, 90, 150+ days. For a selected third-party application, the native app lists the individual users by last sign-in date, indicating which users can be considered for de-provisioning and potential license cost savings.

# Setup


Once the native app is installed, it provides a streamlit in the native app’s `CORE` schema. The app is pre-configured with sample data. The application is fully functional and does not require any further setup.


To use your organization’s SSO sign-in logs, navigate to the "Setup" tab and "Use your data" section. Push the "Setup users" and "Setup sign-ins" buttons to select tables containing your users and sign-in data. The selected tables must contain the following columns:

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


Infostrux provides a free-to-use extract native app "Infostrux MS Graph API Extractor", and a free-to-use loader native app "Infostrux Loader," that can be used to extract and load the data, but their use is optional.


Alternatively, you can instruct the application to use your data programmatically. You can provide the references to the tables that hold users and sign-in data with the following script:

```SQL


use database <app name>;
call configuration.update_reference(
    'users_table', 
    'ADD', 
    SYSTEM$REFERENCE(
        'TABLE', 
        '<database name>.<schema name>.<users table name>', 
        'PERSISTENT', 
        'SELECT'));


call configuration.update_reference(
    'signins_table', 
    'ADD', 
    SYSTEM$REFERENCE(
        'TABLE', 
        '<database name>.<schema name>.<sign-ins table name>', 
        'PERSISTENT', 
        'SELECT'));

```
The script will grant equivalent `SELECT` privileges on the tables to the application.

# Notes


The application can be uninstalled, and all objects owned by it dropped by running:

```SQL
drop application <application_name> cascade;
```


The application may write INFO-level logs to the event table associated with your account.

