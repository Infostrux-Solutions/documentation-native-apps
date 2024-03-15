Infostrux Cost Optimization app
===============================



# Introduction
This application offers fundamental account monitoring, providing insights into storage costs and storage sizes, compute credit usage broken down to credit usage by warehouse, credit usage over time, and users' warehouse usage.


# Description
The application will provide you with information on the overall utilization of your Snowflake credits.
It will also give you details on how much storage you are currently consuming. 
Moreover, the application will present you with a comprehensive analysis of your usage patterns. 
Additionally, you will be able to view information on the current warehouses that are being employed.

This information will help you take the necessary steps to reduce the overall cost of your Snowflake.



# INSTALLATION INSTRUCTIONS
After getting and installing the application, the ACCOUNTADMIN needs to grant it permission to read the SNOWFLAKE database for relevant information. This is achieved by opening a worksheet and executing:
```
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO APPLICATION <application name>;
```
To find the application name, execute:
```
SHOW APPLICATIONS;
```


During application usage, users select a WAREHOUSE, then view a page with all reports.


Your databases remain inaccessible. The application solely accesses Snowflake metadata and usage data.


# UNINSTALL INSTRUCTIONS
```
DROP APPLICATION <application name>;
```

# Contact
For any feedback or questions please contact us at https://infostrux.com