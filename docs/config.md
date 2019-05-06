# Configuration

## Overview

Set by code base at entry point.

The items to be set are the following two and both are required.

* DB connection type
* Delegate for creating a `System.Data.IDbDataParameter` instance

```csharp
ConfigContainer.AddDefault(
    DbConnectionKind.SqlServer, // DB connection type
    () => new SqlParameter()    // Delegate for creating a `System.Data.IDbDataParameter` instance
);
```

!!! note "WCF"
    As the entry point is hidden in WCF, please be sure to write in the place where Service implementation method etc. are called.

## DB connection type

EasySqlParser supports the following DB connections.

|DB connection type|Description|
|:---------------|:---|
|SqlServer|DbConnection is `System.Data.SqlClient.SqlConnection`<br/>**Microsoft SQL Server 2012 or later**
|SqlServerLegacy|DbConnection is `System.Data.SqlClient.SqlConnection`<br/> **Microsoft SQL Server 2008 and earlier**
|Oracle|DbConnection is `Oracle.DataAccess.Client.OracleConnection`<br/>or<br/>`Oracle.ManagedDataAccess.Client.OracleConnection`<br/> **Oracle 12c or later**
|OracleLegacy|DbConnection is `Oracle.DataAccess.Client.OracleConnection`<br/>or<br/>`Oracle.ManagedDataAccess.Client.OracleConnection`<br/> **Oracle 11g and ealier**
|DB2|DbConnection is `IBM.Data.DB2.DB2Connection`
|AS400|DbConnection is `IBM.Data.DB2.iSeries.iDB2Connection`
|MySql|DbConnection is `MySql.Data.MySqlClient.MySqlConnection`
|PostgreSql|DbConnection is `Npgsql.NpgsqlConnection`
|SQLite|DbConnection is `System.Data.SQLite.SQLiteConnection`<br/>or<br/>`Microsoft.Data.Sqlite.SqliteConnection`
|Odbc|DbConnection is `System.Data.Odbc.OdbcConnection`
|OleDb|DbConnection is `System.Data.Odbc.OleDbConnection`

!!! Warning
    Make sure that the delegate for creating the `System.Data.IDbDataParameter` instance is set according to the DB connection implementation

!!! note "In case of multiple DB connection"
    If you need multiple DB connections such as SQL Server and Oracle, please register additional settings using the `AddAditional` method

    ```csharp
    ConfigContainer.AddAdditional(
        DbConnectionKind.Oracle, // DB connection type
        () => new OracleParameter()    // Delegate for creating a `System.Data.IDbDataParameter` instance
    );
    ```

## Cache

Like DOMA, EasySqlParser has a function to cache the contents of read SQL file.

Therefore, if you want to modify only SQL files while the Web application is running, the changes are not reflected because the cache is effective.

In such a case, if you allow `SqlParser.ClearCache(path/to/sqlfile)` to be executed from the system administrator, it is possible to modify only SQL files.

!!! Warning
    Think of the cache clear function as a secondary thing and restart the application server as much as possible.