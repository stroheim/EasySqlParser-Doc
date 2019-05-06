# SQL

## Overview

_2-way-sql_ can do the following by adding it to regular SQL statement

* Assemble dynamic SQL with comments and bind variables
* Execute SQL from tools such as SQL Server Management Studio, SQL Developer, A5: SQL Mk-2, and DBeaver by specifying test data for bind variables

## SQL File

The SQL file mentioned here is a plain text file written in 2-way-sql

### Encoding

SQL file encoding must be UTF-8

## SQL Comment

Perform value binding and conditional branching by writing an expression in the SQL comment.  
SQL comments interpreted by EasySqlParser are called expression comments.

* Bind variable comment
* Embedded variable comment
* Condition comment

!!! Warning
    DOMA `Repeated Comments (for)` and `Column List Expanded Comments (expand, populate)` are not supported by EasySqlParser

### Bind variable comment

An expression comment that indicates a bind variable is called a bind variable comment. Bind variables are generated as `System.Data.IDbDataParameter`.

Bind variables are shown enclosed in block comments `/ * to * /`.  
Bind variable names correspond to object property names.  
The corresponding parameter type must be a basic type such as `string` or `int` or `System.Collections.Generic.IEnumerable<T>`.  
You must specify test data immediately after the bind variable comment.  
However, test data is not used at runtime.

!!! Warning
    Bind variables were fields in DOMA, but are public properties in EasySqlParser

!!! Note
    If there is no test data, an error will occur when it is executed with tools such as SSMS, so test data is required

### Basic type parameters

The following is an example of SqlParser parameters and the corresponding SQL

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
    Unlike DOMA, EasySqlParser **1 bind variable** must also be a property

### Parameters of type IEnumerable&lt;T>

The bind variable passed to the IN clause must be a property of type `IEnumerable <T>`

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

### Embedded variable comment

An expression comment that indicates an embedded variable is called an embedded variable comment.  
Embedded variable values ​​are embedded directly as part of SQL when assembling SQL.

To prevent SQL injection, it is prohibited to include the following values ​​in embedded variable values:

* a ingle quotation (`'`)
* a semi colon (`;`)
* line comment (such as `--`John Doe)
* block comment (such as `/*` John Doe */)

Embedded variables are indicated by a block comment `/* # ～ */`.  
Embedded variable names are mapped to object property names.  
You can use embedded variables when you want to programmatically assemble parts of SQL, such as the ORDER BY clause.

The following is an example of SqlParser parameters and the corresponding SQL

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

### Condition comment

#### if and end

An expression comment that indicates a conditional branch is called a conditional comment.  
The syntax is as follows:

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

This translates to the following SQL statement if `BusinessEntityID` is not `null`

```sql
SELECT
  *
FROM
  HumanResources.Employee
WHERE
  BusinessEntityID = @BusinessEntityID
```

**Parameter markers like `@` are automatically determined by the type of DB connection**

If `BusinessEntityID` is `null`, it will be the following SQL statement

```sql
SELECT
  *
FROM
  HumanResources.Employee
```

As with DOMA, automatic removal of WHERE and HAVING in conditional comments removes the WHERE clause

!!! note "Automatic removal of WHERE and HAVING in conditional comments"
    All conditional comments are false, and the WHERE clause and the HAVING clause do not hold
    WHERE before `/*%if ~ */` is automatically removed

!!! note "Automatic removal of AND and OR in conditional comments"
    If you use conditional comments, it automatically determines the necessity of output for `AND` and `OR` that follow the condition.  
    For example, in the following SQL, `BirthDate` is `null`
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
    The `AND` after `/*% end */` is automatically removed and the following SQL is generated
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

#### elseif and else

You can also use the following syntax for `elseif` or `else` between `/*% if conditionals */` and `/*% end*/`

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

If `BusinessEntityID != Null` holds

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

If `BusinessEntityID == null && DepartmentID != Null` holds

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

If `BusinessEntityID == null && DepartmentID == null` holds

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

#### Nested conditional comments

Conditional comments can be nested, as in .NET syntax

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

!!! Warning "Constraints in conditional comments"
    You can not use `if ~ end` across clauses (SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY, etc.)<br/>
    e.g)
    ```sql
    SELECT
      *
    FROM
      HumanResources.Employee
    /*%if BusinessEntityID != null */
    WHERE BusinessEntityID = /* BusinessEntityID */99
    /*%end*/

    ```
    It is also an error if `if` and `end` are at different levels as shown below
    ```sql
    SELECT
      *
    FROM
      HumanResources.Employee
    WHERE
      BusinessEntityID in /*%if BirthDate != null */(...  /*%end*/ ...)
    ```

#### Repeated comment

Not supported by EasySqlParser

#### Selected column list expansion comment

Not supported by EasySqlParser

#### Update column list generation comment

Not supported by EasySqlParser

#### Normal block comment

If the third character after the `/*` is a character that can not be used at the beginning of a C# identifier(Except for `%`, `#`, `@`, `"` and `'`, which have special meaning in whitespace and expressions), it is considered a normal block comment

Normal block comment example

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

Expression comment example

```sql
/* ～*/
/*a～*/
/*$～*/
/*%～*/
/*#～*/
/*@～*/
/*"～*/
/*'～*/
```

!!! note
    Use `/** ~ */` for normal block comments unless there is a specific reason

#### Normal line comment

`--` is a line comment

EasySqlParser does not interpret line comments