# Practical usage

## Overwview

You need to define _parameter object_ to parse SQL statement

!!! note "Parameter object"
    Although _parameter objects_ may be anonymous types, it is recommended to implement them as explicit types unless there is a specific reason  
    Furthermore, placing the above parameter object class and SQL file close together makes it easier to manage.

!!! Warning
    _Parameter objects_ can not access nested properties

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

    In the above case, EasySqlParser can not access ChildCondition.Name

## Parse result

The result of `SqlParser#Parse` (type `SqlParserResult`) is returned as an object with the following properties:

* ParsedSql
* DebugSql
* DbDataParameters

### ParsedSql

A parsed SQL statement to pass to `ADO.NET`, `Dapper`, `Entity Framework Core`, etc.

Dynamically interpreted by the contents of _parameter object_, SQL statement is assembled
Valid property values ​​are replaced by the parameter name of the `System.Data.IDbDataParameter` instance

### DebugSql

SQL statement for log output

Dynamically interpreted by the contents of _parameter object_, SQL statement is assembled  
It will be a SQL statement with valid property values ​​embedded

### DbDataParameters

A collection of `System.Data.IDbDataParameter` instances for SQL execution

!!! note "System.Data.IDbDataParameter"
    We use `System.Data.IDbDataParameter` for two reasons:

    * Consideration for legacy such as `ADO.NET DataSet`
    * Neutrality considerations from frameworks such as `Dapper` and `Entity Framework Core`

    Due to this, conversion of parameters is required for `Dapper` and` Entity Framework Core`.  
    Please see [Sample](https://github.com/stroheim/EasySqlParser.Examples) for a detailed example.

## Execution example (normal)

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
            DbConnectionKind.SqlServer, // DB connection type
            () => new SqlParameter()    // Delegate for creating a `System.Data.IDbDataParameter` instance
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

In the above case, the results are as follows

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

## Paging

EasySqlParser can generate SQL statement for data acquisition and SQL statement for number acquisition by rewriting SQL file internally

The result of `SqlParser#ParsePaginated` is returned as an object with the following properties:

* Result
* CountResult

### Result

`SqlParserResult` for getting data

It is of type `SqlParserResult` and has　`ParsedSql`, `DebugSql`, `DbDataParameters` as well as `SqlParser#Parse`.

### CountResult

`SqlParserResult` for getting the number

It also has `ParsedSql`, `DebugSql`, `DbDataParameters`

## Execution example (paging)

The SQL file and parameter object are omitted because they are the same

```csharp
public class Program
{
    static void Main(string[] args)
    {
        ConfigContainer.AddDefault(
            DbConnectionKind.SqlServer,
            () => new SqlParameter()
        );
        ConfigContainer.EnableCache = true;

        var condition = new SqlCondition
                {
                    BirthDateFrom = new DateTime(1980, 1, 1),
                    BirthDateTo = new DateTime(1990, 1, 1)
                };
        var parser = new SqlParser("path/to/SelectEmployees.sql", condition);
        var result = parser.ParsePaginated(10, 10); // getting 11 to 20 items
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

## Use of ROW_NUMBER

If an arbitrary string is given to the `rowNumberColumn` parameter of `SqlParser#ParsePaginated`, it will be forced to use the `ROW_NUMBER` function regardless of the DB connection type.

`rowNumberColumn` is a virtual column name that receives the return value of the `ROW_NUMBER` function

Can be used to output line numbers in the View layer

!!! Warning
    Please check if DB can use `ROW_NUMBER` before using  
    The available DBs and versions are as follows:

    |DB        |Version|
    |:---------|:------|
    |SQLServer |2005 |
    |Oracle    |9i   |
    |MySQL     |8    |
    |PostgreSQL|8.4  |
    |DB2       |9.1  |
    |SQLite    |3.25 |

## Execution example (ROW_NUMBER)

The SQL file and parameter object are omitted because they are the same

```csharp
public class Program
{
    static void Main(string[] args)
    {
        ConfigContainer.AddDefault(
            DbConnectionKind.SqlServer,
            () => new SqlParameter()
        );
        ConfigContainer.EnableCache = true;

        var condition = new SqlCondition
                {
                    BirthDateFrom = new DateTime(1980, 1, 1),
                    BirthDateTo = new DateTime(1990, 1, 1)
                };
        var parser = new SqlParser("path/to/SelectEmployees.sql", condition);
        var result = parser.ParsePaginated(10, 10, "LineNo"); // getting 11 to 20 items
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