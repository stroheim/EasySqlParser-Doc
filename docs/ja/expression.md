# 式言語

## 概要

SQL中の式コメントには簡易な式を記述できます。 文法はC#とほとんど同じです。 ただし、C#で可能なことすべてができるわけではありません。

!!! Warning
    DOMAで提供されている下記の機能はEasySqlParserではサポートされていません

    * 算術演算子
    * 連結演算子
    * インスタンスメソッドの呼び出し
    * インスタンスフィールドへのアクセス
    * staticメソッドの呼び出し
    * staticフィールドへのアクセス
    * カスタム関数の使用

## リテラル

EasySqlParserでは以下のリテラルが利用できます

|リテラル|型|
|:------|:--|
|null| - |
|true|bool|
|false|bool|
|10|int|
|10L|long|
|0.123F|float|
|0.123D|double|
|0.123M|decimal|
|10U|uint|
|10UL|ulong|
|"a"|string|

数値の型は、リテラルの最後に `L` や `F` などのサフィックスを付与して区別します。 サフィックスは大文字でなければいけません。

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
  /*%if MiddleNames != null && MiddleNames.Count > 0 */
  t1.MiddleName IN /* MiddleNames */('M')
  /*%end*/
```

## 比較演算子

EasySqlParserでは以下の比較演算子が利用できます

|演算子|説明|
|:-----|:----|
|==|等値演算子|
|!= |不等値演算子[^1]|
|< |小なり演算子|
|<= |小なりイコール演算子|
|> |大なり演算子|
|>= |大なりイコール演算子|

[^1]: 不等演算子は `<>` でも可

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  /*%if BusinessEntityID > 0 */
  BusinessEntityID = /* BusinessEntityID */1
  /*%end*/
```

## 論理演算子

EasySqlParserでは以下の論理演算子が利用できます

|演算子|説明|
|:-----|:----|
|! |論理否定演算子|
|&& |論理積演算子|
| &#124;&#124; |論理和演算子|

括弧を使って、演算子が適用される優先度を制御できます。

```sql
SELECT
  t0.*
FROM
  HumanResources.Employee t0
  INNER JOIN HumanResources.EmployeeDepartmentHistory t1
    ON t0.BusinessEntityID = t1.BusinessEntityID
  INNER JOIN HumanResources.Department t2
    ON t1.DepartmentID = t2.DepartmentID
  INNER JOIN Person.Person t3
    ON t0.BusinessEntityID = t3.BusinessEntityID
WHERE
/*%if (BusinessEntityID == null || DepartmentID == null) && FirstName != null */
  t3.FirstName = /* FirstName */'smith'
/*%end*/
```

## インスタンスプロパティへのアクセス

ドット `.` で区切ってプロパティ名を指定することでインスタンスプロパティにアクセスできます。 可視性はpublicでなければなりません。

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
  /*%if MiddleNames != null && MiddleNames.Count > 0 */
  t1.MiddleName IN /* MiddleNames */('M')
  /*%end*/
```

## 組み込み関数の使用

組み込み関数は、主に、SQLにバインドする前にバインド変数の値を変更するためのユーティリティです。

たとえば、LIKE句で前方一致検索を行う場合に次のように記述できます。

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
  /*%if FirstName != null && FirstName != "" */
  t1.FirstName LIKE /* @StartsWith(FirstName) */'A%'
  /*%end*/
ORDER BY
  t0.BusinessEntityID
```

EasySqlParserでは以下の組み込み関数が利用できます

| 戻り値の型 | 関数名とパラメータ | 概要 |
|:--|:--|:--|
|string | @Escape(string text) | LIKE演算のためのエスケープを行うことを示します。<br/> 戻り値は入力値をエスケープした文字列です。<br/> エスケープにはデフォルトのエスケープ文字（$）を用いて行われます。<br/> 引数にnullを渡した場合、nullを返します。|
|string | @Escape(string text, char escapeChar) | LIKE演算のためのエスケープを行うことを示します。<br/> 戻り値は入力値をエスケープした文字列です。<br/> エスケープは第2引数で指定したエスケープ文字を用いて行われます。<br/> 最初の引数にnullを渡した場合、nullを返します。|
|string | @StartsWith(string text)| 前方一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを後ろに付与した文字列です。<br/> エスケープにはデフォルトのエスケープ文字（$）を用いて行われます。<br/> 引数にnullを渡した場合、nullを返します。|
|string | @StartsWith(string text, char escapeChar)| 前方一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを後ろに付与した文字列です。<br/> エスケープは第2引数で指定したエスケープ文字を用いて行われます。<br/> 最初の引数にnullを渡した場合、nullを返します。|
|string | @Contains(string text)| 中間一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを前と後ろに付与した文字列です。<br/> エスケープはデフォルトのエスケープ文字（$）を用いて行われます。<br/> 引数にnullを渡した場合、nullを返します。|
|string | @Contains(string text, char escapeChar)| 中間一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを前と後ろに付与した文字列です。<br/> エスケープは第2引数で指定したエスケープ文字を用いて行われます。<br/> 最初の引数にnullを渡した場合、nullを返します。|
|string | @EndsWith(string text)| 後方一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを前に付与した文字列です。<br/> エスケープはデフォルトのエスケープ文字（$）を用いて行われます。<br/> 引数にnullを渡した場合、nullを返します。|
|string | @EndsWith(string text, char escapeChar)| 後方一致検索を行うことを示します。<br/> 戻り値は入力値をエスケープしワイルドカードを前に付与した文字列です。<br/> エスケープは第2引数で指定したエスケープ文字を用いて行われます。<br/> 最初の引数にnullを渡した場合、nullを返します。|
|DateTime | @TruncateTime(DateTime dateTime)| 時刻部分を切り捨てることを示します。<br/> 戻り値は時刻部分が切り捨てられた新しい日付です。<br/> 引数にnullを渡した場合、nullを返します。|
|DateTimeOffset | @TruncateTime(DateTimeOffset dateTimeOffset)| 時刻部分を切り捨てることを示します。<br/> 戻り値は時刻部分が切り捨てられた新しい日付です。<br/> 引数にnullを渡した場合、nullを返します。|
