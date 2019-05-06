# 実践的な使い方

## 概要

SQL文をパースするにはパラメータオブジェクトを定義する必要があります

!!! note "パラメータオブジェクト"
    パラメータオブジェクトは匿名型でも構いませんが、特に理由がなければ明示的な型として実装することをお勧めします  
    さらに言えば、上記パラメータオブジェクトクラスとSQLファイルを近くに置くとことで管理しやすくなります

!!! Warning
    パラメータオブジェクトはネストされたプロパティへのアクセスはできません

    ```csharp
    public class SqlCondition
    {
        public ChildCondition ChildCondition { get; set; }
        public string Name { get; set;}
    }

    public class ChildCondition
    {
        public string Name { get; set;}
    }
    ```

    上記の場合、ChildCondition.Nameへはアクセスできません

## パース結果

`SqlParser#Parse` の結果( `SqlParserResult` 型)は下記プロパティを持つオブジェクトとして返されます

* ParsedSql
* DebugSql
* DbDataParameters

### ParsedSql

`ADO.NET`、`Dapper`、`Entity Framework Core` などに渡すためのパースされたSQL文です

パラメータオブジェクトの内容によって動的に解釈されSQL文が組み立てられます  
有効なプロパティの値は `System.Data.IDbDataParameter` インスタンスのパラメータ名によって置き換えられます

### DebugSql

ログ出力のためのSQL文です

パラメータオブジェクトの内容によって動的に解釈されSQL文が組み立てられます  
有効なプロパティの値が埋め込まれた状態のSQL文となります

### DbDataParameters

SQL実行のための `System.Data.IDbDataParameter` インスタンスのコレクションです

!!! note "System.Data.IDbDataParameter"
    `System.Data.IDbDataParameter` を利用しているのは、以下の2つの理由によるものです

    * `ADO.NET DataSet`などレガシー向けへの配慮
    * `Dapper`、`Entity Framework Core` などのフレームワークからの中立性への考慮

    この関係で `Dapper`、`Entity Framework Core` ではパラメータの変換が必要となります  
    詳しい例は[サンプルプログラム](https://github.com/stroheim/EasySqlParser.Examples)をご覧ください

## 実行例(通常)

```sql
/**
SelectEmployees.sql
*/
SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  /*%if MiddleNames != null && MiddleNames.Count > 0 */
  t1.MiddleName IN /* MiddleNames */('M')
  /*%end*/

  /*%if BirthDateFrom != null && BirthDateTo != null */
  AND t0.BirthDate BETWEEN /* BirthDateFrom */'1980-01-01' AND /* BirthDateTo */'1990-01-01'
  /*%end*/

  /*%if FirstName != null && FirstName != "" */
  AND t1.FirstName LIKE /* @StartsWith(FirstName) */'A%'
  /*%end*/
ORDER BY
  t0.BusinessEntityID
```

```csharp
public class SqlCondition
{
    public List<string> MiddleNames { get; set; }
    public DateTime? BirthDateFrom { get; set; }
    public DateTime? BirthDateTo { get; set; }
    public string FirstName { get; set; }
}

public class Program
{
    static void Main(string[] args)
    {
        ConfigContainer.AddDefault(
            DbConnectionKind.SqlServer, // DBコネクションの種類
            () => new SqlParameter()    // SQLパラメータインスタンス作成のデリゲート
        );
        ConfigContainer.EnableCache = true;

        var condition = new SqlCondition
                {
                    BirthDateFrom = new DateTime(1980, 1, 1),
                    BirthDateTo = new DateTime(1990, 1, 1)
                };
        var parser = new SqlParser("path/to/SelectEmployees.sql", condition);
        var result = parser.Parse();
    }
}
```

上記の場合パース結果はそれぞれ下記のとおりです

### SqlParser#Parse ParsedSql

```sql
SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE



   t0.BirthDate BETWEEN @BirthDateFrom AND @BirthDateTo



ORDER BY
  t0.BusinessEntityID
```

### SqlParser#Parse DebugSql

```sql
SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE



   t0.BirthDate BETWEEN '1980/01/01 0:00:00' AND '1990/01/01 0:00:00'



ORDER BY
  t0.BusinessEntityID
```

### SqlParser#Parse DbDataParameters

|index|parameter name|parameter value|
|:----|:-------------|:--------------|
|0    |@BirthDateFrom|new DateTime(1980, 1, 1)|
|1    |@BirthDateTo  |new DateTime(1990, 1, 1)|

## ページング

EasySqlParserが内部でSQLファイルを書き換えることでデータ取得用のSQL文、件数取得用のSQL文を生成できます

`SqlParser#ParsePaginated` の結果は下記プロパティを持つオブジェクトとして返されます

* Result
* CountResult

### Result

データ取得用の `SqlParserResult`

これは `SqlParserResult` 型であり、`SqlParser#Parse` の結果と同様 `ParsedSql`、`DebugSql`、`DbDataParameters` を持っています

### CountResult

件数取得用の `SqlParserResult`

データ取得用と同じく、 `ParsedSql`、`DebugSql`、`DbDataParameters` を持っています

## 実行例(ページング)

SQLファイルやパラメータオブジェクトは同じなので省略

```csharp
public class Program
{
    static void Main(string[] args)
    {
        ConfigContainer.AddDefault(
            DbConnectionKind.SqlServer, // DBコネクションの種類
            () => new SqlParameter()    // SQLパラメータインスタンス作成のデリゲート
        );
        ConfigContainer.EnableCache = true;

        var condition = new SqlCondition
                {
                    BirthDateFrom = new DateTime(1980, 1, 1),
                    BirthDateTo = new DateTime(1990, 1, 1)
                };
        var parser = new SqlParser("path/to/SelectEmployees.sql", condition);
        var result = parser.ParsePaginated(10, 10); // 11件目から20件目を取得
    }
}
```

### SqlParser#ParsePaginated Result.ParsedSql

```sql
SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN @BirthDateFrom AND @BirthDateTo
  

  
ORDER BY
  t0.BusinessEntityID
 offset 10 rows fetch next 10 rows only
```

### SqlParser#ParsePaginated Result.DebugSql

```sql
SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN '1980/01/01 0:00:00' AND '1990/01/01 0:00:00'
  

  
ORDER BY
  t0.BusinessEntityID
 offset 10 rows fetch next 10 rows only
```

### SqlParser#ParsePaginated CountResult.ParsedSql

```sql
select count(*) from ( SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN @BirthDateFrom AND @BirthDateTo
  

  
) t_
```

### SqlParser#ParsePaginated CountResult.DebugSql

```sql
select count(*) from ( SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN '1980/01/01 0:00:00' AND '1990/01/01 0:00:00'
  

  
) t_
```

## ROW_NUMBER の使用

`SqlParser#ParsePaginated` の `rowNumberColumn` パラメータに任意の文字列を与えると、DB接続の種類を無視して強制的に `ROW_NUMBER` 関数を使うようになります

`rowNumberColumn` は `ROW_NUMBER` 関数の戻り値を受け取る仮想列名です

View 層で行番号を出力する場合に利用できます

!!! Warning
    DBが `ROW_NUMBER` を利用できるかどうか確認したうえで利用してください  
    利用可能なDBとバージョンは下記のとおりです

    |DB        |Version|
    |:---------|:------|
    |SQLServer |2005 |
    |Oracle    |9i   |
    |MySQL     |8    |
    |PostgreSQL|8.4  |
    |DB2       |9.1  |
    |SQLite    |3.25 |

## 実行例(ROW_NUMBER)

SQLファイルやパラメータオブジェクトは同じなので省略

```csharp
public class Program
{
    static void Main(string[] args)
    {
        ConfigContainer.AddDefault(
            DbConnectionKind.SqlServer, // DBコネクションの種類
            () => new SqlParameter()    // SQLパラメータインスタンス作成のデリゲート
        );
        ConfigContainer.EnableCache = true;

        var condition = new SqlCondition
                {
                    BirthDateFrom = new DateTime(1980, 1, 1),
                    BirthDateTo = new DateTime(1990, 1, 1)
                };
        var parser = new SqlParser("path/to/SelectEmployees.sql", condition);
        var result = parser.ParsePaginated(10, 10, "LineNo"); // 11件目から20件目を取得
    }
}
```

### ROW_NUMBER Result.ParsedSql

```sql
select * from ( select temp_.*, row_number() over( ORDER BY
  temp_.BusinessEntityID
 ) as LineNo from ( SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN @BirthDateFrom AND @BirthDateTo
  

  
) as temp_ ) as temp2_ where LineNo > 10 and LineNo <= 20
```

### ROW_NUMBER Result.DebugSql

```sql
select * from ( select temp_.*, row_number() over( ORDER BY
  temp_.BusinessEntityID
 ) as LineNo from ( SELECT
    t0.BusinessEntityID
  , t1.FirstName
  , t1.MiddleName
  , t1.LastName
  , t0.BirthDate
  , t0.MaritalStatus
  , t0.Gender
  , t0.HireDate
FROM HumanResources.Employee t0
INNER JOIN Person.Person t1
  ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
  

  
   t0.BirthDate BETWEEN '1980/01/01 0:00:00' AND '1990/01/01 0:00:00'
  

  
) as temp_ ) as temp2_ where LineNo > 10 and LineNo <= 20
```