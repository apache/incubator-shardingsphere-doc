+++
pre = "<b>3.6. </b>"
title = "Test Engine"
weight = 6
chapter = true
+++

ShardingSphere provided a full functionality test engine. it defines SQLs in xml files, every single SQL is drove by SQL parse unit test engine and integration test engine,
each engine is suit for H2、MySQL、PostgreSQL、SQLServer and Oracle

The SQL parsing unit test covers both SQL placeholder and literal dimension. 
Integration test can be further divided into two dimensions of strategy and JDBC; the former one includes strategies as Sharding, table Sharding, database Sharding, and read-write split while the latter one includes `Statement` and `PreparedStatement`.

Therefore, one SQL can drive 5 kinds of database parsing * 2 kinds of parameter transmission modes + 5 kinds of databases * 5 kinds of Sharding strategies * 2 kinds of JDBC operation modes = 60 test cases, to enable ShardingSphere to achieve the pursuit of high quality.

## Integration Test Engine

### Configuration Files

In order to make test engine easier to setup, integration-test is designed to modify the following configuration files to execute all assertions without any `Java` code modification:

  - environment type
    - /incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/env.properties
    - /incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/env/`SQL-TYPE`/dataset.xml
    - /incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/env/`SQL-TYPE`/schema.xml
  - test case type
    - /incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/cases/`SQL-TYPE`/`SQL-TYPE`-integrate-test-cases.xml
    - /incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/cases/`SQL-TYPE`/dataset/`SHARDING-TYPE`/*.xml
  - sql-case 
    - /incubator-shardingsphere/sharding-sql-test/src/main/resources/sql/sharding/`SQL-TYPE`/*.xml

#### Environment Configuration

Integration test depends on existed database environment, developer need to setup the configuration file for corresponding database to test: 

Firstly, setup configuration file `/incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/env.properties`, for example: 

```properties
# the switch for PK, concurrent, column index testing and so on
run.additional.cases=false

# sharding rule, could define multiple rules
sharding.rule.type=db,tbl,dbtbl_with_masterslave,masterslave

# database type, could define multiple databases(H2,MySQL,Oracle,SQLServer,PostgreSQL)
databases=MySQL,PostgreSQL

# MySQL configuration
mysql.host=127.0.0.1
mysql.port=13306
mysql.username=root
mysql.password=root

## PostgreSQL configuration
postgresql.host=db.psql
postgresql.port=5432
postgresql.username=postgres
postgresql.password=

## SQLServer configuration
sqlserver.host=db.mssql
sqlserver.port=1433
sqlserver.username=sa
sqlserver.password=Jdbc1234

## Oracle configuration
oracle.host=db.oracle
oracle.port=1521
oracle.username=jdbc
oracle.password=jdbc
```

Secondly, setup configuration file `/incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/env/SQL-TYPE/dataset.xml`. 
Developer can set up metadata and expected data to start the data initialization in `dataset.xml`. For example: 

```xml
<dataset>
    <metadata data-nodes="tbl.t_order_${0..9}">
        <column name="order_id" type="numeric" />
        <column name="user_id" type="numeric" />
        <column name="status" type="varchar" />
    </metadata>
    <row data-node="tbl.t_order_0" values="1000, 10, init" />
    <row data-node="tbl.t_order_1" values="1001, 10, init" />
    <row data-node="tbl.t_order_2" values="1002, 10, init" />
    <row data-node="tbl.t_order_3" values="1003, 10, init" />
    <row data-node="tbl.t_order_4" values="1004, 10, init" />
    <row data-node="tbl.t_order_5" values="1005, 10, init" />
    <row data-node="tbl.t_order_6" values="1006, 10, init" />
    <row data-node="tbl.t_order_7" values="1007, 10, init" />
    <row data-node="tbl.t_order_8" values="1008, 10, init" />
    <row data-node="tbl.t_order_9" values="1009, 10, init" />
</dataset>
```

Developer can customize DDL to create databases and tables in `schema.xml`.

#### SQL Configuration

After setup environment and initial data, developer need to define SQL test cases.
the SQL to be asserted in file `/incubator-shardingsphere/sharding-sql-test/src/main/resources/sql/sharding/SQL-TYPE/*.xml`. For example: 

```xml
<sql-cases>
    <sql-case id="update_without_parameters" value="UPDATE t_order SET status = 'update' WHERE order_id = 1000 AND user_id = 10" />
    <sql-case id="update_with_alias" value="UPDATE t_order AS o SET o.status = ? WHERE o.order_id = ? AND o.user_id = ?" db-types="MySQL,H2" />
  </sql-cases>
```

Developer setup the SQL for assertion and database type during on the configuration file. And these SQLs could share in different test engine, that's why we extract the `sharding-sql-test` as a stand alone module.

#### Assertion Configuration

We have confirmed what kind of sql execute in which environment in upon config, hereby let's define the data for assert.
There are two kinds of config for assert, one is at `/incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/cases/SQL-TYPE/SQL-TYPE-integrate-test-cases.xml`.
This file just like an index, defined the sql, parameters and expected index position for execution. the SQL is the value for `sql-case-id`. For example: 

```xml
<integrate-test-cases>
    <dml-test-case sql-case-id="insert_with_all_placeholders">
       <assertion parameters="1:int, 1:int, insert:String" expected-data-file="insert_for_order_1.xml" />
       <assertion parameters="2:int, 2:int, insert:String" expected-data-file="insert_for_order_2.xml" />
    </dml-test-case>
</integrate-test-cases>
```

Another kind of config for assert is the data, as known as the corresponding expected-data-file in SQL-TYPE-integrate-test-cases.xml, which is at `/incubator-shardingsphere/sharding-integration-test/sharding-jdbc-test/src/test/resources/integrate/cases/SQL-TYPE/dataset/SHARDING-TYPE/*.xml`.  
This file is very like the dataset.xml we mentioned before, and the difference is that expected-data-file contains some other assert data, such as the return value after a sql execution. For examples:  

```xml
<dataset update-count="1">
    <metadata data-nodes="db_${0..9}.t_order">
        <column name="order_id" type="numeric" />
        <column name="user_id" type="numeric" />
        <column name="status" type="varchar" />
    </metadata>
    <row data-node="db_0.t_order" values="1000, 10, update" />
    <row data-node="db_0.t_order" values="1001, 10, init" />
    <row data-node="db_0.t_order" values="2000, 20, init" />
    <row data-node="db_0.t_order" values="2001, 20, init" />
</dataset>
```
so far, all config files are ready, we just need to launch the corresponding test case. we don't need to modify any Java code, just set up some config files.
this will reduce the difficulty for ShardingSphere testing

### Notice

1. If Oracle needs to be tested, please add Oracle driver dependencies to the pom.xml.
1. 10 splitting-databases and 10 splitting-tables are used in the integrated test to ensure the test data is full, so it will take a relatively long time to run the test cases.

## SQL Parsing Engine Test

### Prepare Data

Not like Integration test, SQL parse test doesn't need a specific database environment, we just need to define the sql we want to parse, and the assert data:

#### SQL Data

We mentioned `sql-case-id` in Integration test，test-case-id could be shared in different module to test, and the file is at `/incubator-shardingsphere/sharding-sql-test/src/main/resources/sql/sharding/SQL-TYPE/*.xml` 

#### Parser Assert Data

The assert data is at `/incubator-shardingsphere/sharding-core/sharding-core-parse/sharding-core-parse-test/src/test/resources/sharding/SQL-TYPE/*.xml`
n that xml file, we could assert against the table name, token or sql condition and so on. For example:

```xml
<parser-result-sets>
<parser-result sql-case-id="insert_with_multiple_values">
        <tables>
            <table name="t_order" />
        </tables>
        <tokens>
            <table-token start-index="12" table-name="t_order" length="7" />
        </tokens>
        <sharding-conditions>
            <and-condition>
                <condition column-name="order_id" table-name="t_order" operator="EQUAL">
                    <value literal="1" type="int" />
                </condition>
                <condition column-name="user_id" table-name="t_order" operator="EQUAL">
                    <value literal="1" type="int" />
                </condition>
            </and-condition>
            <and-condition>
                <condition column-name="order_id" table-name="t_order" operator="EQUAL">
                    <value literal="2" type="int" />
                </condition>
                <condition column-name="user_id" table-name="t_order" operator="EQUAL">
                    <value literal="2" type="int" />
                </condition>
            </and-condition>
        </sharding-conditions>
    </parser-result>
</parser-result-sets>
```

When these configs are ready, we could launch the test engine in sharding-core-parse-test to test SQL parse. 