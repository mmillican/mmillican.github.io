---
layout: post
title: 'Querying data with Dapper for a Kendo UI Grid'
subtitle: 'A short tutorial on using Dapper to query filtered & paginated data for a Kendo Grid'
date: 2017-01-23 12:00:00
author: 'Matt Millican'
---

[Dapper](https://github.com/StackExchange/dapper-dot-net) is a small library that allows you to execute SQL queries and map the results to strongly typed (or dynamic) objects by extending your existing IDbConnection.  For the past several years, I've used EntityFramework as my ORM, and while I still think it's a great tool, there are times when you may want to have more control over the SQL you're executing, namely for performance.

Unfortunately, when using Dapper, or the standard SqlDataReader, you lose one of the key benefits of Entity Framework which is [automatic] deferred execution of things like filtering, sorting and pagination from LINQ statements thanks to the `IQueryable<T>` interface.  For example, with Entity Framework, the following LINQ statement, the SQL query

```c#
var users = _dbContext.Users.OrderByDescending(x => x.CreatedOn).Take(25);
```

would result in a query similar to

```sql
SELECT TOP 25 * FROM Users ORDER BY CreatedOn DESC
```

When using Dapper (or SqlDataReader), you are responsible for handing filtering, pagination and sorting.

One of the benefits of using [Kendo UI's ASP.NET server-side wrappers](http://demos.telerik.com/aspnet-mvc/)  when using Entity Framework is it too will defer execution of functions to the database.  So how do we get the benefit of smaller query results when using the Kendo Grid and Dapper?  Let's take a look.

### Setting up Dapper

To install Dapper, simply run the following command from your Package Manager Console in Visual Studio (alternatively, you can use the UI to search for and install Dapper)

```
Install-Package Dapper
```

Once installed, Dapper is fairly easy to set up as it just extends `IDbConnection` and simply requires the connection to be open (it will fail if it's not).  In my latest projects, I'm using Dapper side by side with EF, so in my `DbContext` class, I have something that looks like this:

``` c#
public IEnumerable<T> Query<T>(string sql, dynamic param = null)
{
    // Database.Connection is inherited from DbContext
    return Database.Connection.Query<T>(sql, param as object,
        commandTimeout: Database.CommandTimeout);
}

public T QueryFirst<T>(string sql, dynamic param = null)
{
    return Database.Connection.QueryFirst<T>(sql, param as object,
        commandTimeout: Database.CommandTimeout);
}

public T QueryFirstOrDefault<T>(string sql, dynamic param = null)
{
    return Database.Connection.QueryFirstOrDefault<T>(sql, param as object,
        commandTimeout: Database.CommandTimeout);
}
```

Now that we have Dapper set up, we can implement our Grid's code and a controller action method to retrieve the data

### Implementing the Kendo Grid and controller

For this example, let's say we want to display a list of users in a simple grid that just has pagination enabled (filtering will be covered in a future post).  Our user model looks like this:

``` c#
public class User 
{
    public int Id { get; set; }

    public string Email { get; set; }

    public string FirstName { get; set; }
    public string LastName { get; set; }

    public bool IsDisabled { get; set; }
}
```

To keep things simple, we'll use the following for our grid definition:

``` html
@(Html.Kendo().Grid<UserModel>()
    .Name("user-grid")
    .DataSource(ds => ds.Ajax()
        .Read("GetUsers", "Users")
        .PageSize(20))
    .Pageable()
    // For brevity, we'll let Kendo automatically display columns
)
```

Now, for the fun part of wiring up the grid to retrieve data.  For purposes of demonstration, I'm going to assume not using Dependency Injection and that you already have an initialized DbContext in your controller

``` c#
public ActionResult GetUsers([DataSourceRequest] DataSourceRequest request)
{
    var sql = @"
        SELECT 
        *
        FROM (
            SELECT
                ROW_NUMBER() OVER (ORDER BY LastName, FirstName ASC) AS Row
                ,Id
                ,LastName
                ,FirstName
                ,Email
                ,IsDisabled
            FROM Users 
            ORDER BY LastName, FirstName ASC
        ) u WHERE u.Row > @start and u.Row <= @end";

    // Get the current paging info from the request to determine start/end
    var start = (request.Page - 1) * request.PageSize;
    var end = request.Page * request.PageSize;

    var users = _dbContext.Query<User>(sql, new {start, end});
    var userCount = _dbContext.QueryFirst<int>("SELECT COUNT(1) FROM Users");

    var result = new DataSourceResult 
    {
        Data = users,
        Total = userCount
    };

    return Json(result);
}
```

Let's take a look at what we're doing here.  To start, we have a simple SQL statement that simply selects all the users from our table, and uses the ROW_NUMBER() function to get the row number within that result set.  This will make it easier to do paging.

When using the server side wrappers for Kendo, we can use the DataSourceRequest object to retrieve "meta data" about the request, which includes things like current page, page size, filtering criteria, etc.  For simplicity sake, we're only using request.Page and request.PageSize.  With those, we're doing some simple math to figure out the start and the end range for our current page of the result set and passing those into the query as parameters.

Normally, you'd have some WHERE clauses in there, so you would want to apply those to the queries (including the count query directly below to get the total result count).  There's another Nuget package, called `[Dapper.SqlBuilder](https://www.nuget.org/packages/Dapper.SqlBuilder/)` which adds some "templating" options to Dapper queries.  

Finally, we're creating a DataSourceResult with our data and total page size to return the data the grid needs.  If we were using Entity Framework and `IQueryable<T>`, you could do `users.ToDataSourceResult(request)` to get the same result (and would defer execution).

For more information and documentation on Kendo's server-side wrappers and data binding, [check out the documentation](http://docs.telerik.com/aspnet-mvc/introduction).

It may be a little more difficult to write your own SQL, but the payoff is worth it. You’ll get much better performance overall and there will be less headaches down the road. I know there’s a lot more to this topic, but hopefully this gives you a head start so you can begin using Kendo Grid and Dapper to retrieve data.  I plan to cover filtering and sorting in future posts.

Feel free to leave any thoughts or comments on querying data for Kendo Grids (or other controls) below.