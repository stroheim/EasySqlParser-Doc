# 設定

## 概要

エントリーポイントでコードベースによる設定を行います

設定する項目は以下の2つでどちらも必須です

* DB接続の種類
* `System.Data.IDbDataParameter` インスタンスを作成するためのデリゲート

```csharp
ConfigContainer.AddDefault(
    DbConnectionKind.SqlServer, // DBコネクションの種類
    () => new SqlParameter()    // SQLパラメータインスタンス作成のデリゲート
);
```

!!! note "WCF"
    WCFではエントリーポイントが隠ぺいされているので、Service実装メソッドなど必ず呼ばれる場所に記述してください

## DB接続の種類

EasySqlParserでは以下のDB接続がサポートされています

|DB接続の種類|説明|
|:---------------|:---|
|SqlServer|DB接続の実装は`System.Data.SqlClient.SqlConnection`<br/>**Microsoft SQL Server 2012 以降**
|SqlServerLegacy|DB接続の実装は`System.Data.SqlClient.SqlConnection`<br/> **Microsoft SQL Server 2008 以前**
|Oracle|DB接続の実装は`Oracle.DataAccess.Client.OracleConnection`<br/>または<br/>`Oracle.ManagedDataAccess.Client.OracleConnection`<br/> **Oracle 12c 以降**
|OracleLegacy|DB接続の実装は`Oracle.DataAccess.Client.OracleConnection`<br/>または<br/>`Oracle.ManagedDataAccess.Client.OracleConnection`<br/> **Oracle 11g 以前**
|DB2|DB接続の実装は`IBM.Data.DB2.DB2Connection`
|AS400|DB接続の実装は`IBM.Data.DB2.iSeries.iDB2Connection`
|MySql|DB接続の実装は`MySql.Data.MySqlClient.MySqlConnection`
|PostgreSql|DB接続の実装は`Npgsql.NpgsqlConnection`
|SQLite|DB接続の実装は`System.Data.SQLite.SQLiteConnection`<br/>または<br/>`Microsoft.Data.Sqlite.SqliteConnection`
|Odbc|DB接続の実装は`System.Data.Odbc.OdbcConnection`
|OleDb|DB接続の実装は`System.Data.Odbc.OleDbConnection`

!!! Warning
    `System.Data.IDbDataParameter` インスタンスを作成するためのデリゲートはDB接続の実装に応じたものを設定するようにしてください

!!! note "複数DB接続の場合"
    SQL ServerとOracleなど複数のDB接続が必要な場合は`AddAditional`メソッドを使って追加の設定を登録してください

    ```csharp
    ConfigContainer.AddAdditional(
        DbConnectionKind.Oracle, // DBコネクションの種類
        () => new OracleParameter()    // SQLパラメータインスタンス作成のデリゲート
    );
    ```

## キャッシュ

EasySqlParserではDOMAと同様、読み込んだSQLファイルの内容をキャッシュする機能を搭載しています。

このため、Webアプリケーション稼働中にSQLファイルのみの修正を行いたい場合、キャッシュが効いているため変更が反映されません。

このような場合には `SqlParser.ClearCache(path/to/sqlfile)` をシステム管理者から実行できるようにしておけば、SQLファイルのみの修正が可能となります。

!!! Warning
    キャッシュクリア機能は補助的なものと考え、可能な限りアプリケーションサーバを再起動してください