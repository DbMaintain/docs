---
title: "Command Line"
date: 2019-01-11T17:47:21+01:00
draft: false
---

To launch DbMaintain from the command line, create a properties file `dbmaintain.properties` that configures the location of your scripts and the target database:
	
```properties
database.driverClassName=org.hsqldb.jdbcDriver
database.url=jdbc:hsqldb:mem:testdb
database.userName=sa
database.password=

# Comma-separated list of all database schemas used.
database.schemaNames=PUBLIC
```

Make sure you've correctly set `JAVA_HOME` to your JRE or J2SDK home directory, and set `DBMAINTAIN_HOME` to the installation directory of dbmaintain. Set `DBMAINTAIN_JDBC_DRIVER` to the jar file that contains the necessary JDBC driver for your DBMS, or edit the file `bin/setJdbcDriver.sh` (or .bat) to set `JDBC_DRIVER` to the correct value. Then execute following command (Replace .sh by .bat if you're using windows)
	
```bash
/path/to/dbmaintain/dbmaintain.sh updateDatabase path/to/scriptFolderOrArchive
```

To use a different config file than `dbmaintain.properties`, add the parameter -f followed by the config file name. System properties can also be used to configure the command line.

See the [ant tasks](../ant-tasks) page for more info on the available operations. The full list of configuration properties can be found on the [configuration](../configuration) page.