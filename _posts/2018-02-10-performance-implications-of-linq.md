---
layout: post
title: Performance Implications of Linq
---

Linq (language-integrated query) was released as part of .NET Framework 3.5 (C# 3.0) back in 2007 with the intention of making code more readable and to allow developers to create applications without learning more than one programming language (e.g SQL). The idea would be that C# developers would be able to perform more complex queries on their dataset without writing complex SQL queries.

To paint a simple example, say we wanted to get all of my posts in the last week:

```SQL
SELECT Id, Name, Created FROM Posts WHERE Created > DATEADD(week,-1,GETDATE()) AND Name = 'Gary';
```
But with Linq:

```C#
var posts = //some SQL: (SELECT * FROM Posts;)
var myPosts = posts.Where(x => x.Created > DateTime.Now.AddDays(-7) && x.Name = "Gary");
```
If I then wanted to get John's posts in the last 2 weeks:

```C#
var johnsPosts = posts.Where(x => x.Created > DateTime.Now.AddDays(-14) && x.Name = "John");
```
This caught on quite quickly - we can do 2 queries and only hit the DB once! Awesome right!?

**No.**

## Why not?
To understand why this isn't awesome, and is probably the source of some of the major performance issues your .NET applications we will jump in to the source of .NET Core (note: .NET Framework is the same, but we can view the source of .NET Core).

GitHub: [https://github.com/dotnet/corefx/blob/master/src/System.Linq/src/System/Linq/Where.cs#L60-L75](https://github.com/dotnet/corefx/blob/master/src/System.Linq/src/System/Linq/Where.cs#L60-L75)

```C#
private static IEnumerable<TSource> WhereIterator<TSource>(IEnumerable<TSource> source, Func<TSource, int, bool> predicate)
{
	int index = -1;
	foreach (TSource element in source)
	{
	    checked
	    {
		index++;
	    }

	    if (predicate(element, index))
	    {
		yield return element;
	    }
	}
}
```
### So what is happening here?
First of all, when we perform a wide `SELECT` statement on a database table we are loading all of the results in to memory, this will immediately consume memory resources for data - most of which you know you are never going to use.

Secondly we are then looping over every single row returned from the database and performing an `if` condition on it.

Without access to Linq - this is the code we would write to acheive the same result:

```C#
List<Post> posts = //SELECT * FROM Posts;
List<Post> garysPosts = new List<Post>();
foreach(Post post in posts)
{
    if (post.Name == "Gary" && post.Created > DateTime.Now.AddDays(-7))
    {
	    garysPosts.Add(post);
    }
}
```    
*note: This would actually be quicker than Linq. Linq adds even more overhead than this foreach*

> ...Do not use LINQ if you care much about performance. - [codymanix](https://stackoverflow.com/a/3156074/179450)

So far we have acheived the following:

 1. Loaded more rows than required from the database - using up resources.
 2. Performed an inefficient loop over those items which gets exponentially slower based on the increasing number of rows.
 3. Copied some of those rows in to a new list - taking up more resources.

Excessive use of this pattern leads to slow response times, platform slowdown and ultimately service outages as you run out of memory.

## How do we fix it?

Easy - use SQL instead of Linq.

By "use SQL" I don't mean go and install Entity Framework or some other ORM, but actually write your SQL queries to only fetch the information you need from the database. Databases like MSSQL are incredibly powerful and efficient tools, don't be scared to test their limits.

I personally use NPoco to write and execute SQL queries from my .NET applications. It has .NET Framework and .NET Standard (Core) support and maps results to your models automatically, but anything that lets you write raw SQL without interference will work fine.

NPoco: https://github.com/schotime/NPoco

## Summary

This is the most common performance pitfalls I see in .NET applications. It's quite common for us as developers to use a library that makes something easier and assume it's doing magic under the covers, just keep in mind that .NET is built on the same C# toolset that you and I work with every day, there is no magic fairy dust!

If this is something that you've seen yourself do in the past or are experiencing some performance issues in a .NET application containing queries like this, I challenge you stop yourself from writing these Linq queries and start writing better defined SQL - I can guarantee both you and your clients will notice the difference.
