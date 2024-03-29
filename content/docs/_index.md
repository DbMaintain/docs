---
title: "Documentation"
date: 2019-01-11T17:47:21+01:00
draft: false
---
	
* [Script organization](#script-organization)
    * [Incremental and repeatable scripts](#incremental-and-repeatable-scripts)
    * [Postprocessing scripts](#postprocessing-scripts)
	* [Multi-database / user support](#multi-database-user-support)
	* [Patches](#patches)
	* [Qualifier inclusion / exclusion](#qualifier_inclusion__exclusion)
	  
* [The DBMAINTAIN_SCRIPTS table](#the-dbmaintain-scripts-table)
    * [Error handling](#error-handling)
    * [Setting a baseline revision](#setting-a-baseline-revision)
    * [Configure and run DbMaintain](#configure-and-run-dbmaintain)
        * [From the command line](#from-the-command-line)
        * [Using ant](#using-ant)
        * [From Java code](#from-java-code)
	  
* [DbMaintain Operations](#dbmaintain-operations)
    * [Create a script archive](#create-a-script-archive)
    * [Update the database](#update-the-database)
    * [Mark error script performed](#mark-error-script-performed)
    * [Mark error script reverted](#mark-error-script-reverted)
    * [Mark the database as up-to-date](#mark-the-database-as-up-to-date)
    * [Check script updates](#check-script-updates)
    * [Clear the database](#clear-the-database)
    * [Clean the database](#clean-the-database)
    * [Disable constraints](#disable-constraints)
    * [Update sequences](#update-sequences)

* [Preserve database objects](#preserve-database-objects)
* [PL/SQL support](#plsql-support)
* [Native runner support](#native-runner-support)
	
Script Organization
---------------------

Database scripts have to be organized in a folder structure, like demonstrated in following example:
	
    scripts/01_v1.0/01_products_and_orders.sql
                    02_users.sql
            02_v1.1/01_add_barcode_column.sql
                    02_drop_itemcode_column.sql

Folder and script names must start with an index number followed by an underscore and a description. The index indicates the sequence of the scripts or script folders.

Suppose you add a new script called `03_add_productid_sequence.sql`. The next time you call the `updateDatabase` operation, the database is updated incrementally by executing this new script. Unless you use the `fromScratch` option, updating an already executed incremental script is not allowed: the updateDatabase action will fail if you try to do so. If the `fromScratch` option is used and an existing incremental script is changed, the database is cleared (i.e. all database objects are dropped) and all scripts are executed again.
	
### Incremental and repeatable Scripts

The database scripts from the example above are incremental: each script contains a `delta`, and should be executed only once in the proper sequence. But DbMaintain also supports repeatable scripts: A repeatable script must be written in such a way that it can be executed multiple times. For example, function or stored procedure defitions and views can be organized as repeatable scripts. The advantage of a repeatable script is that it can be modified: When changed, the script is simply executed again. 
	
You can recognize repeatable scripts from the file name: it has no index number. The directories that contain repeatable scripts also cannot have an index number. If you put a non-indexed script inside an indexed one, dbmaintain will give an error.
	
Repeatable scripts are always executed after all incremental (indexed) scripts. Therefore, they cannot be located inside an indexed folder. If your database project contains repeatable scripts, it's a good idea to clearly separate them from the incremental scripts, like in following example:
	
    scripts/incremental/01_v1.0/01_products_and_orders.sql
                                02_users.sql
                        02_v1.1/01_add_barcode_column.sql
                                02_drop_itemcode_column.sql
            repeatable/proc_purge_old_orders.sql
                    proc_refresh_materialized_views.sql

	
### Postprocessing Scripts
Some projects have scripts that need to be executed each time a script was executed, such as for example a script that compiles all stored procedures. Post processing scripts can be defined for this purpose. Post processing scripts have to be located in the directory called `postprocessing`, located directly in the root of the configured scripts folder. They can be indexed, but they don't have to be: Indexes can be used to indicate the execution sequence of the post-processing scripts. An indexed post-processing script can be modified without causing an error or triggering a from-scratch update. If a post-processing script is modified, all of them are executed again. For example:
	
    scripts/incremental/01_v1.0/01_products_and_orders.sql
                                02_users.sql
                        02_v1.1/01_add_barcode_column.sql
                                02_drop_itemcode_column.sql
            repeatable/proc_purge_old_orders.sql
                    proc_disactivate_inactive_users.sql
            postprocessing/01_compile_all.sql
                        02_grant_select_to_read_user.sql

### Scripts Archives
In the above examples we showed that DbMaintain can execute scripts from folders. It is also possible to package these scripts in a jar and then execute the scripts from this jar. This way you can treat scripts as a real artifact in the same way as you would for example deliver an application in an EAR.

A typical setup would be to let your build system (e.g. Hudson) create a scripts archive during the build along with all the other artifacts. The same scripts archive can then be rolled out on your development, system test, acceptance and production environments. Just use the updateDatabase task and DbMaintain will examine the current state of the target database and execute the appropriate scripts on them.

Use the createScriptArchive task to create a scripts jar. You can then specify the jar as script location for the other tasks.
See the [ant-tasks](./ant-tasks) and [maven-goals](./maven-goals) for examples and more info on the available parameters.


### Multi-Database / User Support
On some projects, database scripts need to be executed using different database connections. For instance, if your database consists of multiple schemas and you need to log in with a different user to be able to perform modifications in another schema. 
	
You can configure multiple databases, each of them identified with a different logical name. This logical name can be used in the script name to indicate the target database of the script. If the script name doesn't indicate a target database, it will be executed on the default database.
	
The target database can be indicated in the script name using an @ sign. For incremental scripts the target database is separated from the index with an underscore; repeatable script names start with the target database indication.
	
For example, suppose we have a second schema called `USERS`, and we have to login with a different user to create or alter tables. We have configured a second database connection identified by the logical name `users`. The scripts 02_users.sql and proc_disactivate_inactive_users.sql have to be executed with this other user:
	
    scripts/incremental/01_v1.0/01_products_and_orders.sql
                                02_@users_users.sql
                        02_v1.1/01_add_barcode_column.sql
                                02_drop_itemcode_column.sql
            repeatable/proc_purge_old_orders.sql
                    @users_proc_disactivate_inactive_users.sql
            postprocessing/01_compile_all.sql
                        02_grant_select_to_read_user.sql


### Patches
Because of the incremental nature of database updates, newly added scripts always have to come last: If a script is added with a lower index number than one that has already been executed, DbMaintain will give an error (or recreate the database from scratch, if this option is enabled). 
	
However it sometimes occurs that, while a new version of an application is being developed, an existing release of the application has to be patched, and this patch involves an incremental database change. In this case, a new incremental script must be created that precedes the scripts from the current development version. 
	
For example: Suppose that our current production version is v1.0, and we're developing version 1.1. A fix must be applied in production that involves a database change: for this we created the script 03_add_product_status.sql. This fix is also merged into our current development branch, which leads us to the following sequence of scripts:
	
    scripts/incremental/01_v1.0/01_products_and_orders.sql
                                02_@users_users.sql
                                03_add_product_status.sql
                        02_v1.1/01_add_barcode_column.sql
                                02_drop_itemcode_column.sql

When we try to deploy to a test database on which the scripts in folder 02_v1.1 were already rolled out, DbMaintain will give an error. If the `fromScratchEnabled` option is set to true, the database will automatically be recreated from scratch. This will cause all data to be lost, which is in most cases not desirable since the database may be used by the customer to test the newly developed features.
	
To be able cope with such situations, we offer the notion of a `patch`. A script or a folder containing scripts can be marked as being a `patch`, by adding the #patch qualifier to the file or folder name. A patch has to be written in such a way that it can be executed on both the current production version and the latest development version. This will lead to the following scripts:
	
    scripts/incremental/01_v1.0/01_products_and_orders.sql
                                02_@users_users.sql
                                03_#patch_add_product_status.sql
                        02_v1.1/01_add_barcode_column.sql
                                02_drop_itemcode_column.sql
	
DbMaintain offers an option called `allowOutOfSequenceExecutionOfPatches`. If set to `true`, DbMaintain will start executing all patch scripts that weren't executed yet, before executing any other scripts. Note that you should never enable this option when deploying to the production database!
	
The #patch token in the script file name is in fact a script qualifier, which can also be used for other purposes. The following section discusses the usage of qualifiers for inclusion and exclusion of scripts.
	
### Qualifier Inclusion / Exclusion
Scripts can be annotated with qualifiers by including # + the qualifier name in the filename. These qualifiers can be used to in- or exclude scripts from execution. This can serve a number of purposes: a typical use-case is to qualify scripts as containing test-data and exclude them from execution when updating a production database. Scripts can contain multiple qualifiers. The #patch indication as described in the previous section is also a qualifier - but with a special meaning.
	
Qualifiers can also be used when different DBMS-es are supported. For example:

        01_#refdata_#postgres_products.sql

This is a reference data script to be executed only on postgres databases.
	
Inclusion or exclusion of qualified scripts can be performed using the properties excludedQualifiers and includedQualifiers. Multiple qualifiers can be specified, separated by commas. If you don't specify anything for the includedQualifiers property, all scripts are executed unless excluded by using the excludedQualifiers property. If includedQualifiers is not empty, scripts are not executed unless they have at least one included qualifier. When included qualifiers are specified, scripts without a qualifier will never be executed. 	
All qualifiers have to be declared using the qualifiers property (to avoid problems due to typing errors).
	
Following example uses ant to execute scripts that are targeted for postgres or for every supported dbms (qualifier everydbms), while excluding all refdata files:

```xml
<updateDatabase scriptLocations="${database.archive}" qualifiers="postgres,mysql,everydbms,refdata" includedQualifiers="postgres,everydbms" excludedQualifiers="refdata">
    <database driverClassName="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:mem:mydb" userName="admin" password="pwd" schemaNames="PUBLIC" />
</updateDatabase>
```

When using a properties file to configure dbmaintain (e.g. for command-line use), the properties to use are dbMaintainer.qualifiers, dbmaintainer.includedQualifiers and dbmaintainer.excludedQualifiers.


The DBMAINTAIN_SCRIPTS Table
------------------------------
The database keeps track of all executed scripts in the table DBMAINTAIN_SCRIPTS. The table definition looks like the following:
	
|FILE_NAME|Relative path of the script file, starting from the root of the scripts directory or archive.|
|---------|----------------------------------|
|FILE_LAST_MODIFIED_AT|Last modification timestamp of the script. If the property useScriptFileLastModificationDates is set to true, this timestamp is used to determine whether a script might have changed, in order to improve performence when there are many or large scripts. If the timestamp is unchanged, we assume that the script didn't change, and the script doesn't have to be read to calculate the checksum. If the timestamp changed, the checksum is verified to decide whether the script changed.|
|CHECKSUM|MD5 hash calculated on the contents of the script. This value is used to determine whether a script has changed since the last update.|
|EXECUTED_AT|Indicates when the script was executed on the database. This is info for the user, which is not used by the system.|
|SUCCEEDED|Indicates whether the script was executed successfully (1 if successful, 0 if there was an error)|


Error handling
----------------
If an error occurs during the execution of a script, DbMaintain immediately stops execution and performs a rollback of the script.
If a repeatable script caused the error you can simply fix this repeatable script and it will be executed again when executing updateDatabase the next time.

If an incremental script caused the error and you try to perform the updateDatabase operation again, DbMaintain will refuse to do it and will just give an error.
This is because implicit or explicit commits could have been performed by the script itself. For example a create table statement in oracle implicitly performs a commit.
As a result the database could be in an invalid state and should first be cleaned up. There are 2 options to recover from this state:

* Fix the script, manually perform the changes of the script and call the markErrorScriptPerformed task.
* Fix the script, revert committed changes of the script (if any) and call the markErrorScriptReverted task.
    
The markErrorScriptPerformed task will just flag the script as successful. The next updateDatabase will then start with the next script.
The markErrorScriptReverted task will remove the failed record from the scripts table. The next updateDatabase will then re-start with that script again.

Note: if the from scratch option is enabled, you can simply fix the script. The next updateDatabase will then automatically re-create the database from scratch.


Setting a baseline revision
-----------------------------
Sometimes you don't start from an empty database. After a release for example, you could do an export of the database and use that image as a starting point for other databases.

DbMaintain supports setting a baseline revision. If you set this revision, all scripts with a lower version will no longer be taken into account.
Modifying or removing them will no longer give an error. You could for example cleanup the old scripts or replace the old scripts with the export script.

If you create a scripts archive, it will only contain scripts starting from this revision. In other words, it will only contain the deltas with the image. This results in smaller jar files that only contain the actual changes for the next release.

Note: when a scripts folder or a script does not have a version, the version is set to the value 'x'.
For example, suppose you have the following structure:

    scripts/01_release_1/001_my_script.sql

Then the version number of this script will be x.1.1. If you want to set this script as the first script of the baseline, you will have to set the baseline revision to x.1.1 and not to 1.1.


Configure and run DbMaintain
------------------------------
DbMaintain operations can be executed in various ways: from the command line, using ant or directly from Java code. In a future release, maven integration will be provided.
	 
### From the command line
Launch scripts are provided for Windows and *nix to perform operations from the command line. To be able to use them, you have to make following preparations:

* Make sure `JAVA_HOME` is set to the installation directory of your JRE or JDK. (Must be Java version 5 or higher)	
* Make sure `DBMAINTAIN_HOME` is set to the installation directory of dbmaintain.
* Set `DBMAINTAIN_JDBC_DRIVER` to the jar file that contains the necessary JDBC driver for your DBMS, or edit the file `bin/setJdbcDriver.bat` or `bin/setJdbcDriver.sh` to set `JDBC_DRIVER` to the correct value.

For instance, to create an archive file containing all scripts simply call (Replace .sh by .bat if you're using windows):

    /path/to/dbmaintain/bin/dbmaintain.sh createScriptArchive path/to/archive path/to/scriptFolder
	 
To bring a database up-to-date, execute following command:
	 
    /path/to/dbmaintain/bin/dbmaintain.sh update path/to/scriptFolderOrArchive

If a file named dbmaintain.properties is available in the execution directory, this file is automatically loaded. To load another file, add -config path/to/configFile to the command.
	
You can get an overview of all available command line operations and their usage by simply executing `dbmaintain.sh/bat` with no arguments.
	 
### Using ant
Ant tasks are provided for all DbMaintain operations. To be able to use these tasks, you have to declare them in your build file, e.g. as follows:
	 
```xml
<path id="dbmaintain-lib">
    <fileset dir="${dbmaintain.home}/lib">
    <include name="*.jar"/></fileset>
</path>
<taskdef resource="dbmaintain-anttasks.xml" classpathref="dbmaintain.lib"/>
```

You can perform a database update with the following task.
	 
```xml
<updateDatabase scriptLocations="${database.archive}">
    <database driverClassName="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:mem:mydb" userName="admin" password="pwd" schemaNames="PUBLIC" />
</updateDatabase>
```

For an overview of all ant tasks with all their attributes, refer to the [ant-tasks](./ant-tasks) page.

### From Java Code
To launch DbMaintain operations from Java code, first create an instance of `org.dbmaintain.MainFactory` and then use that factory to create the DbMaintain, ConstraintsDisabler, etc instances.

```java
URL configurationUrl = new File("dbmaintain.properties").toURI().toURL();
MainFactory mainFactory = new MainFactory(configurationUrl);
DbMaintainer dbMaintainer = createDbMaintainer()
```

To perform a database update simply call:

```java	
dbMaintainer.updateDatabase(false);
```

### Configure the Database(s)

DbMaintain can be configured with one or more databases. If you use more than one database, the target database has to be specified in the script (see [Multi-database / user support](#multi-database-user-support)). To configure a single database in a properties file, add the following properties:

```properties
database.driverClassName=<fully qualified JDBC driver class name>
database.url=<database URL>
database.userName=<database username>
database.password=<database password>
database.schemaNames=<comma separated list of all database schemas used>
```

If you configure multiple databases, you have to give a logical name to each of them, and list them in the property databases.names. The logical name has to be added to the property name to configure the driver, url, etc. The logical name can also be used in the script filenames to indicate the target database. The database that is listed first automatically becomes the default database, which is the target database for scripts that don't specify one. For example, if we want to configure databases called users and orders:
	
```properties
databases.names=users,orders
database.users.driverClassName=<fully qualified JDBC driver class name>
database.orders.driverClassName=<fully qualified JDBC driver class name>
...
```

If, for a certain property, all databases share the same value, you can use the default property name to configure it, e.g. if all databases use the hsqdb database driver. If you have one database but you need to connect using different database users to execute certain scripts, you can configure multiple databases that share all properties except their credentials. E.g.:
	
```properties
databases.names=admin,user,read
database.driverClassName=oracle.jdbc.driver.OracleDriver
database.url=jdbc:oracle:thin://mydb:1521:MYDB
database.admin.username=admin
database.admin.password=adminpwd
database.admin.schemaNames=admin
database.user.userName=user
database.user.password=userpwd
database.user.schemaNames=user
database.read.userName=read
database.read.password=readpwd
database.read.schemaNames=read
```

If you use ant, you can configure the database(s) with a `database` subelement like follows:
	
```xml
    <database driverClassName="<fully qualified JDBC driver class name>" url="<database URL>" userName="admin" password="pwd" schemaNames="<comma separated list of all database schemas used>" />
```

or with multiple `database` subelements having a `name` attribute:
	
```xml
    <database name="<logical database name>" driverClassName="<fully qualified JDBC driver class name>" url="<database URL>" userName="admin" password="pwd" schemaNames="<comma separated list of all database schemas used>" />
```

In some cases, you want to configure a database with a certain name but exclude it from execution. The scripts that have an excluded database as target will simply be ignored. This can be useful if you only want to execute the scripts with a certain target database on an end-to-end test database but not on a local test datatabase. When using command line configuration, simply add a property database.`logicalname`.included=false. When using ant, create a `database` element with a `name` attribute and an `included` attribute set to `false`.


DbMaintain Operations
-----------------------
Whether you launch DbMaintain from the command line, using ant or directly from Java code, the same set of operations is exposed. See the [ant-tasks](./ant-tasks) and [maven-goals](./maven.goals) for examples and more info on the available parameters.


Create a script archive
=========================
The simplest way to use DbMaintain is to simply configure it with a scripts folder. However when building the project, it's a good idea to also package the scripts in the form of an archive file that you can publish as a build artifact.


Update the Database
===================
Updates the database to the latest version. First it checks which scripts were already applied to the database and executes the new scripts or the updated repeatable scripts. If an existing incremental script was changed, removed, or if a new incremental script has been added with a lower index than one that was already executed, an error is given; unless the `fromScratch` option is enabled: in that case all database objects are removed and the database is rebuilt from scratch. If there are post-processing scripts, these are always executed at the end.


Mark error script performed
=============================
Task that indicates that the failed script was manually performed. The script will NOT be run again in the next update. No scripts will be executed by this task.


Mark error script reverted
=========================
Task that indicates that the failed script was manually reverted. The script will be run again in the next update. No scripts will be executed by this task.


Mark the database as up-to-date
=========================
Updates the state of the database to indicate that all scripts have been executed, without actually executing them. This can be useful when you want to start using DbMaintain on an existing database, or after having fixed a problem directly on the database.


Check Script Updates
=========================
Performs a dry-run of the `updateDatabase` operation and prints all detected script updates, without executing anything. This operation fails whenever the `updateDatabase` operation would fail, i.e. if there are any irregular script updates and `fromScratchEnabled` is `false` or if a patch script was added out-of-sequence and `allowOutOfSequenceExecutionOfPatches` is `false`. An automatic test could be created that executes this operation against a test database that cannot be updated from scratch, to enforce at all times that no irregular script updates are introduced.


Clear the Database
=========================
This operation removes all database objects from the database, such as tables, views, sequences, synonyms and triggers. The database schemas will be left untouched: this way, you can immediately start an update afterwards. This operation is also called when a from-scratch update is performed. The table dbmaintain_scripts is not dropped but all data in it is removed. It's possible to exclude certain database objects to make sure they are not dropped, like described in [Preserve database objects](#preserve-database-objects).


Clean the Database
=========================
If you want to remove all existing data from the tables in your database, you can call the cleanDatabase operation. The data from the table dbmaintain_script is not deleted. It's possible to preserve data from certain tables, like described in [Preserve database objects](#preserve-database-objects). The updateDatabase operation offers an option to automatically clean the database before doing an update.


Disable Constraints
=========================
Disables all foreign key, not null and unique constraints. The updateDatabase operation offers an option to automatically disable the constraints after the scripts were executed: This can be useful for a database used to write persistence layer unit tests, to simplify the definition and limit the necessary amount of test data. When using the automatic database update option of [unitils](http://www.unitils.org), which uses DbMaintain, the disable constraints option is enabled by default.


Update Sequences
=========================
This operation is also mainly useful for automated testing purposes. This operation sets all sequences and identity columns to a minimum value. By default this value is 1000, but is can be configured with the `lowestAcceptableSequenceValue` option. The updateDatabase operation offers an option to automatically update the sequences after the scripts were executed.


Preserve Database Objects
=========================
It's possible to exclude certain database objects from being dropped when a fromScratch update occurs, or when the clearDatabase operation is invoked: the data in these tables is also not removed when performing an update using the cleanDatabase option. If you want a table to be dropped in case of a fromScratch update, but you want it's data to be preserved when performing the cleanDatabase operation, you can use one of the preserveDataOnly properties.
	
```properties
# Comma separated list of database items that may not be dropped or cleared by DbMaintain when
# updating the database from scratch.
# Schemas can also be preserved entirely. If identifiers are quoted (eg "" for oracle) they are considered
# case sensitive. Items may be prefixed with the schema name. Items that do not have a schema prefix are 
# considered to be in the default schema.
dbMaintainer.preserve.schemas=
dbMaintainer.preserve.tables=
dbMaintainer.preserve.views=
dbMaintainer.preserve.materializedViews=
dbMaintainer.preserve.synonyms=
dbMaintainer.preserve.sequences=

# Comma separated list of table names. The tables listed here will not be emptied during a cleanDatabase operation.
# Data of the dbmaintain_scripts table is preserved automatically.
# Tables listed here will still be dropped before a fromScratch update. If this is not desirable
# you should use the property dbMaintainer.preserve.tables instead.
# Schemas can also be preserved entirely. If identifiers are quoted (eg "" for oracle) they are considered 
# case sensitive. Items may be prefixed with the schema name. Items that do not have a schema prefix are considered 
# to be in the default schema
dbMaintainer.preserveDataOnly.schemas=
dbMaintainer.preserveDataOnly.tables=
```
    
Note that the items in the preserve properties must exist! If one of them doesn't exist, the operation is aborted and an error message is given. This makes sure that there weren't any typos and that something is dropped accidently.
	
Note that in the ant tasks, no attributes are provided for configuring items to preserve. You have to refer to a properties file from the ant task by using the configFile attribute.


PL/SQL support
=========================
Oracle, DB2 and MySQL PL-SQL syntax is supported (eg. functions and stored procedures). Blocks of PL/SQL code must always end with a separate line containing a single forward slash.

```sql
DROP FUNCTION MyFunction();

CREATE FUNCTION MyFunction()
BEGIN
END;
/

CREATE OR REPLACE MyFunction2()
BEGIN
END;
/
```

Tip: It's a good idea to use repeatable scripts for your stored procedure definitions: this way, you can simply make changes to these definitions when necessary (use the CREATE OR REPLACE syntax to make sure these scripts are repeatable). Create a postprocessing script that performs a compile of all stored procedures after each database update.


Native Runner Support
=========================
By default DbMainain uses JDBC to execute the scripts. For Oracle and Db2 it is also possible to use the native script runners instead of JDBC. For Oracle you can use `Sql*Plus` and for Db2 you can use the `db2 CLP` runner.
The advantage is that you can use all the commands that these programs support. The disadvantage is that you lose platform independence: these programs need to be installed on the system.

To use one of these runners you have to change the script runner implementation as follows:

```properties
# Oracle
org.dbmaintain.script.runner.ScriptRunner.factory=org.dbmaintain.scriptrunner.SqlPlusScriptRunnerFactory
# Db2
org.dbmaintain.script.runner.ScriptRunner.factory=org.dbmaintain.scriptrunner.Db2ScriptRunnerFactory
```

By default DbMaintain will look in the path for an executable named sqlplus for the SqlPlus script runner and db2 for the Db2 script runner. You can explicitly define the command by setting following properties.

```properties
dbMaintainer.sqlPlusScriptRunner.sqlPlusCommand=sqlplus
dbMaintainer.db2ScriptRunner.db2Command=db2
```    
