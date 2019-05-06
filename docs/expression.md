# Expression language

## Overview

Simple expressions can be written in expression comments in SQL.  
The syntax is almost the same as C#.  
However, not all C# can do.

!!! Warning
    The following functions provided by DOMA are not supported by EasySqlParser.

    * Arithmetic operators
    * String concatenation operator
    * Calling instance methods
    * Accessing to instance fields
    * Calling static methods
    * Accessing to static fields
    * Using custom functions

## Letarl

The following literals are available in EasySqlParser

|Literal|Type|
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

Numeric types are distinguished by adding a suffix such as `L` or `F` to the end of the literal.  
The suffix must be in upper case.

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

## Comparison operator

The following comparison operators are available in EasySqlParser

|Operator|Description|
|:-----|:----|
|==|Equality operator|
|!= |Inequality operator[^1]|
|< |Less than operator|
|<= |Less than or equal operator|
|> |Greater than operator|
|>= |Greater than or equal operator|

[^1]: Can be `<>`

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

## Logical operator

The following logical operators are available in EasySqlParser

|Operator|Description|
|:-----|:----|
|! |Logical negation operator|
|&& |Conditional logical AND operator|
| &#124;&#124; |Conditional logical OR operator|

You can use parentheses to control the priority to which the operator is applied.

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

## Access instance properties

Instance properties can be accessed by specifying property names separated by a dot `.`. Visibility must be public.

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

## Use of built-in functions

Built-in functions are primarily utilities for changing the value of bind variables before binding to SQL.

For example, when performing a forward match search with the LIKE clause, you can write:

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

The following built-in functions are available in EasySqlParser

| Return type | Function name and parameters | Description |
|:--|:--|:--|
|string | @Escape(string text) | Indicates to escape for LIKE operation. <br/>The return value is a string with the input value escaped. <br/>The escape is done using the default escape character ($). <br/> If you pass null as an argument, it returns null.|
|string | @Escape(string text, char escapeChar) | Indicates to escape for LIKE operation. <br/> The return value is a string with the input value escaped. <br/> Escape is performed using the escape character specified in the second argument. <br/> If you pass null as the first argument, it returns null.|
|string|@StartsWith(string text)| Indicates to perform a forward match search. <br/> The return value is a string after escaping the input value and appending a wildcard. <br/> The escape is done using the default escape character ($). <br/> If you pass null as an argument, it returns null.|
|string|@StartsWith(string text, char escapeChar)| Indicates to perform a forward match search. <br/> The return value is a string after escaping the input value and appending a wildcard. <br/> Escape is performed using the escape character specified in the second argument. <br/> If you pass null as the first argument, it returns null.|
|string|@Contains(string text)| Indicates that an intermediate match search is to be performed. <br/> The return value is a string with the input value escaped and wildcards given before and after. <br/> Escape is done using the default escape character ($). <br/> If you pass null as an argument, it returns null.|
|string|@Contains(string text, char escapeChar)| Indicates that an intermediate match search is to be performed. <br/> The return value is a string with the input value escaped and wildcards given before and after. <br/> Escape is performed using the escape character specified in the second argument. <br/> If you pass null as the first argument, it returns null.|
|string|@EndsWith(string text)| Indicates to perform a backward match search. <br/> The return value is a string with the input value escaped and preceded by a wildcard. <br/> Escape is done using the default escape character ($). <br/> If you pass null as an argument, it returns null.|
|string|@EndsWith(string text, char escapeChar)| Indicates to perform a backward match search. <br/> The return value is a string with the input value escaped and preceded by a wildcard. <br/> Escape is performed using the escape character specified in the second argument. <br/> If you pass null as the first argument, it returns null.|
|DateTime|@TruncateTime(DateTime dateTime)| Indicates to truncate the time part. <br/> The return value is a new date with the time portion truncated. <br/> If you pass null as an argument, it returns null.|
|DateTimeOffset|@TruncateTime(DateTimeOffset dateTimeOffset)| Indicates to truncate the time part. <br/> The return value is a new date with the time portion truncated. <br/> If you pass null as an argument, it returns null.|
