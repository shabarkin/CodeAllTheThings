# .NET

# Summary

## Access Controls & Authorization

> You can also use the `AllowAnonymous` attribute to allow access by non-authenticated users to individual actions.
> 
> [https://docs.microsoft.com/en-us/aspnet/core/security/authorization/simple?view=aspnetcore-6.0](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/simple?view=aspnetcore-6.0)
> 

Walk through all the files within the `Controllers` folder and identify all `[AllowAnonymous]` attributes on the endpoint (function) or even controller (class) level. The controller on the Class level can have `[Authorize]` attribute, what means all the endpoints will require authorization, however the `[AllowAnonymous]` attribute on the function/endpoint level will rewrite this permission. 

```bash
grep -HanriEA2 "\[AllowAnonymous\]"
```

```bash
gf -save cs-unauthenticated -HanriEA2 "\[AllowAnonymous\]"
```

## SQL Injections

Identify the pattern and the way the application creates and keeps the connection with the database by reviewing at least one case. Presence of "`using System.Data.SqlClient`" namespace will tell you what type of SQL class an application uses.

**Identify SQL namespace:**

```sql
grep -HanriE "using System.Data.SqlClient"
```

```sql
gf -save cs-sql-namespace -HanriE "using System.Data.SqlClient"
```

**Search all entries of the following .NET SQL classes:** `SqlCommand`, `OracleCommand`, `OdbcCommand`, `OleCommand`

```sql
grep -HanriE "SqlCommand\(|OracleCommand\(|OdbcCommand\(|OleCommand\("
```

```sql
gf -save cs-sql-class -HanriE "SqlCommand\(|OracleCommand\(|OdbcCommand\(|OleCommand\("
```

**Search all entries of the abstract class** `SqlHelper`:

```sql
grep -HanriE "SqlHelper\."
```

```sql
gf -save cs-sql-class-helper -HanriE "SqlHelper\."
```

The common ways to execute SQL query use the `SqlConnection` and `SqlCommand` classes. `SqlCommand` has `ExecuteNonQuery` and [`ExecuteQuery`](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.executenonquery.aspx) methods. NonQuery returns the number of rows that were affected while `ExecuteQuery` returns a DataReader used to read the SQL data stream. You'll be able to recognize all kinds of SQL injection possibilities (format strings, concatenations, interpolations.) when you look at the `ExecuteNonQuery` ,  `ExecuteQuery` and other methods:

1. `EntityCommand`
2. `ExecuteNonQuery`
3. `ExecuteQuery`
4. `ExecuteDataset`
5. `ExecuteScalar`

**Search all entries of the following Sql execution methods:** 

```sql
grep -HanriE "EntityCommand\(|ExecuteNonQuery\(|ExecuteQuery\(|ExecuteDataset\(|ExecuteScalar\("
```

```sql
gf -save cs-sql-exec -HanriE "EntityCommand\(|ExecuteNonQuery\(|ExecuteQuery\(|ExecuteDataset\(|ExecuteScalar\("
```

**Search for the** `CommandType.Text` **directive to identify the places where the SQL query will be processed as the raw string:**

```sql
grep -HanriE "CommandType\.Text"
```

```sql
gf -save cs-sql-text -HanriE "CommandType\.Text"
```
