# SQL

!!! Note

    * DOMA経験者はWarningトピックに注目しながら流し読みしてください
    * DOMAは未経験だが、S2Dao・S2JdbcなどDOMA以外の2-way-sqlライブラリ経験者は全体を流し読みしてください

## 概要

2-way-sqlは通常のSQL文に一味加えることで以下のことが可能となります

* コメントとバインド変数で動的なSQLの組み立てる
* バインド変数に対するテスト用データを指定することでSQL Server Management Studio、SQL Developer、A5:SQL Mk-2、DBeaverなどといったツールからSQLを実行する

## SQLファイル

ここで言うSQLファイルとは2-way-sqlで書かれたプレーンなテキストファイルのことです

### エンコーディング

SQLファイルのエンコーディングはUTF-8でなければいけません

## SQLコメント

SQL コメント中に式を記述することで値のバインディングや条件分岐を行います。  
EasySqlParserによって解釈されるSQLコメントを 式コメント と呼びます。

式コメントは以下のものがサポートされています

* バインド変数コメント
* 埋め込み変数コメント
* 条件コメント

!!! Warning
    DOMAの`繰り返しコメント(for)`、`カラムリスト展開コメント(expand, populate)`はEasySqlParserではサポートされていません

### バインド変数コメント

バインド変数を示す式コメントを バインド変数 コメントと呼びます。 バインド変数は、`System.Data.IDbDataParameter`として生成されます。

バインド変数は `/*～*/` というブロックコメントで囲んで示します。 バインド変数の名前はオブジェクトのプロパティ名に対応します。対応するパラメータの型は`string`、`int`などの基本型 もしくは その `System.Collections.Generic.IEnumerable<T>`でなければいけません。バインド変数コメントの直後にはテスト用データを指定する必要があります。 ただし、テスト用データは実行時には使用されません。

!!! Warning
    バインド変数はDOMAではフィールドでしたが、EasySqlParserではパブリックなプロパティです

!!! Note
    テスト用データがなければSSMS等のツールで実行した際にエラーとなるためテスト用データが必要となります

### 基本型のパラメータ

以下がSqlParserのパラメータと対応するSQLの例です

```csharp
var parser = new SqlParser(
  "path/to/sql",
  new {BusinessEntityID = 99999}
);
var result = parser.Parse();
Console.WriteLine(result.DebugSql);
/*
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID = 99999
*/
```

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID = /* BusinessEntityID */1
```

!!! Warning
    DOMAと異なり、EasySqlParserでは**1つのバインド変数**でもプロパティでなければなりません

### IEnumerable&lt;T&gt;型のパラメータ

IN句に渡すバインド変数は`IEnumerable<T>`型のプロパティである必要があります

```csharp
var parser = new SqlParser(
  "path/to/sql",
  new {BusinessEntityIDs = new List<int>{10, 11, 12}}
);
var result = parser.Parse();
Console.WriteLine(result.DebugSql);
/*
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID IN (10, 11, 12)
*/
```

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID IN /* BusinessEntityIDs */(1)
```

### 埋め込み変数コメント

埋め込み変数を示す式コメントを埋め込み変数コメントと呼びます。 埋め込み変数の値は SQL を組み立てる際に SQL の一部として直接埋め込まれます。

SQL インジェクションを防ぐため、埋め込み変数の値に以下の値を含めることは禁止しています。

* シングルクォテーション
* セミコロン
* 行コメント
* ブロックコメント

埋め込み変数は `/*#～*/` というブロックコメントで示します。 埋め込み変数の名前は オブジェクトのプロパティ名にマッピングされます。 埋め込み変数は ORDER BY 句など SQL の一部をプログラムで組み立てたい場合に使用できます。

以下がSqlParserのパラメータと対応するSQLの例です

```csharp
var parser = new SqlParser(
  "path/to/sql",
  new {
    BusinessEntityID = 1000,
    Orderby = "ORDER BY BirthDate ASC"
  }
);
var result = parser.Parse();
Console.WriteLine(result.DebugSql);
/*
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID > 1000
*/
```

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID > /* BusinessEntityID */1
/*# Orderby */
```

### 条件コメント

#### ifとend

条件分岐を示す式コメントを条件コメントと呼びます。構文は次のとおりです。

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  /*%if BusinessEntityID != null */
  BusinessEntityID = /* BusinessEntityID */1
  /*%end*/
```

これは `BusinessEntityID` が `null` でない場合、以下のSQL文に変換されます

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID = @BusinessEntityID
```

**`@` のようなパラメータマーカーはDB接続の種類で自動的に決定されます**

`BusinessEntityID` が `null` の場合は下記のSQL文になります

```sql
SELECT
  *
FROM
  HumanResources.Employee
```

DOMA同様、条件コメントにおけるWHEREやHAVINGの自動除去機能により、WHERE句は削除されます

!!! note "条件コメントにおけるWHEREやHAVINGの自動除去"
    条件コメントがすべて偽となり、WHERE句やHAVING句が成り立たない場合
    `/*%if ～*/` の前の WHERE は自動で除去されます

!!! note "条件コメントにおけるANDやORの自動除去"
    条件コメントを使用した場合、条件の後ろにつづく `AND` や `OR` について自動で出力の要/不要を判定します。 たとえば、次のようなSQLで `BirthDate` が `null` の場合、
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
      /*%if BirthDate != null */
      t0.BirthDate > /* BirthDate */'1980-01-01'
      /*%end*/
      AND t1.FirstName LIKE 'A%'
    ORDER BY
      t0.BusinessEntityID

    ```
    `/*%end*/`の後ろの`AND`は自動で除去され、下記のSQLが生成されます
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
      t1.FirstName LIKE 'A%'
    ORDER BY
      t0.BusinessEntityID
    ```

#### elseifとelse

`/*%if 条件式*/` と `/*%end*/` の間では、 `elseif` や `else` を表す次の構文も使用できます

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN HumanResources.EmployeeDepartmentHistory t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
  INNER JOIN HumanResources.Department t2
    ON t1.DepartmentID = t2.DepartmentID
WHERE
/*%if BusinessEntityID != null */
  t0.BusinessEntityID = /* BusinessEntityID */9999
/*%elseif DepartmentID != null */
  AND t2.DepartmentID = /* DepartmentID */99
/*%else*/
  AND t1.EndDate IS NULL
/*%end*/
```

`BusinessEntityID != null` が成立する場合

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN HumanResources.EmployeeDepartmentHistory t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
  INNER JOIN HumanResources.Department t2
    ON t1.DepartmentID = t2.DepartmentID
WHERE
  t0.BusinessEntityID = @BusinessEntityID
```

`BusinessEntityID == null && DepartmentID != null` が成立する場合

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN HumanResources.EmployeeDepartmentHistory t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
  INNER JOIN HumanResources.Department t2
    ON t1.DepartmentID = t2.DepartmentID
WHERE
  t2.DepartmentID = @DepartmentID
```

`BusinessEntityID == null && DepartmentID == null` が成立する場合

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN HumanResources.EmployeeDepartmentHistory t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
  INNER JOIN HumanResources.Department t2
    ON t1.DepartmentID = t2.DepartmentID
WHERE
  t1.EndDate IS NULL
```

#### ネストした条件コメント

.NETの構文と同様に、条件コメントはネストさせることができます

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN Person.Person t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
WHERE
/*%if BusinessEntityID != null */
  t0.BusinessEntityID = /* BusinessEntityID */1
  /*%if MiddleName != null */
    AND t1.MiddleName = /* MiddleName */'hoge'
  /*%else*/
    AND t1.MiddleName IS NULL
  /*%end*/
/*%end*/
```

!!! Warning "条件コメントにおける制約"
    節(SELECT,FROM,WHERE,GROUP BY, HAVING,ORDER BYなど)をまたがって`if~end`を使用することはできません<br/>
    例)
    ```sql
    SELECT
      *
    FROM
      HumanResources.Employee
    /*%if BusinessEntityID != null */
    WHERE BusinessEntityID = /* BusinessEntityID */99
    /*%end*/

    ```
    下記のように`if`と`end`が異なるレベルにある場合もエラーとなります
    ```sql
    SELECT
      *
    FROM
      HumanResources.Employee
    WHERE
      BusinessEntityID in /*%if BirthDate != null */(...  /*%end*/ ...)
    ```

#### 繰り返しコメント

EasySqlParserではサポートされていません

#### 選択カラムリスト展開コメント

EasySqlParserではサポートされていません

#### 更新カラムリスト生成コメント

EasySqlParserではサポートされていません

#### 通常のブロックコメント

`/*` の直後に続く3文字目がC#の識別子の先頭で使用できない文字 （ただし、空白および式で特別な意味をもつ `%`、`#`、 `@`、 `"`、 `'` は除く）の場合、 それは通常のブロックコメントだとみなされます

通常のブロックコメント例

```sql
/**～*/
/*+～*/
/*=～*/
/*:～*/
/*;～*/
/*(～*/
/*)～*/
/*&～*/
```

式コメント例

```sql
/* ～*/ ...--3文字目が空白であるため式コメントです。
/*a～*/ ...--3文字目がC#の識別子の先頭で使用可能な文字であるため式コメントです。
/*$～*/ ...--3文字目がC#の識別子の先頭で使用可能な文字であるため式コメントです。
/*%～*/ ...--3文字目が条件コメントの始まりを表す「%」であるため式コメントです。
/*#～*/ ...--3文字目が埋め込み変数コメントを表す「#」であるため式コメントです。
/*@～*/ ...--3文字目が組み込み関数もしくを表す「@」であるため式コメントです。
/*"～*/ ...--3文字目が文字列リテラルの引用符を表す「"」であるため式コメントです。
/*'～*/ ...--3文字目が文字リテラルの引用符を表す「'」であるため式コメントです。
```

!!! note
    特に理由が無ければ通常のブロックコメントは`/**～*/`を利用してください

#### 通常の行コメント

`--` は行コメントです

EasySqlParserは行コメントを解釈しません