---
title: Dashing, how's our little C# ORM doing?
---

It's been almost 5 years since the inception of Dashing so it's due a thorough review, or at least a summation of all my current thoughts regarding it.

To that end I'd like to look back at why it was created in the first place, highlight some of the successes and then take a look to the future.

# Inception

The start of Dashing co-incided with the formation of my current company and really came about as a result of several things:

- A poor selection of .Net ORMs being available at the time
- A "new" thing called Dapper, that had been released from Stackoverflow
- A hope that I(we) could do better

In truth, I'd already had a couple of goes at doing something similar (one of which may still be in use and I can only apologise for) and felt that I was on the verge of something great.

## The others...

The .Net ORM landscape at the time was fairly small ... you had:

- Entity Framework - maybe v4 or v5 at this point and just not up to the job.
- Linq to Sql - dying.
- NHibernate - sooo buggy using the LINQ provider (as in I had 0 confidence that it would return the correct entities).
- LLBLGEN - actually very good, having used it in anger, but not free.
- Dashing - The "MicroORM", very good at hydrating entities from SQL but not overly productive.

There were others sure, but mostly small things with dodgy APIs that didn't really make sense.

## The vision for Dashing

The vision for Dashing has been written at the top of the Readme from day 1:

> It aims to be a strongly typed data access layer that is built with productivity and performance in mind.

To me this meant: can we build a set of tools that allows us, as developers, to be as productive as possible without compromising on the performance of the code TOO much? Specifically, could we:

- minimise configuration as much as possible - it should just work with [POCOs](https://en.wikipedia.org/wiki/Plain_old_CLR_object) i.e. no types, annotations, having to make things virtual.
- provide a simple, strongly typed, API for access i.e. no strings and typed foreign keys.
- automate schema migrations
- make use of Dapper for entity hydration
- generate code, and cache it, at runtime to make it go fast!

I think all of these guidelines have driven the direction that Dashing has gone in. However, I've also always had a presentation, by Jimmy Bogard, ever present on my desktop, that's massively helped inform this: [ORMs, you're doing it wrong](/downloads/Orms.pptx). In that, Jimmy outlines his views on what would make a good ORM and while Dashing doesn't fulfil all of those viewpoints (yet) I don't think it does anything that would offend him!

## An example

The [background](http://polylytics.github.io/dashing/background) page of the documentation goes in to more detail around some of these points but, to illustrate the point, I'd like to copy the two examples from there. Both of these bits of code get a blog post out of a database, as well as the user that authored the blog post. In Dapper, that means a multi-mapping query.

__Using Dapper__
```
return dapperConn.Query<Post, User, Post>(
	// Sql query
	@"select 
		t.[PostId], t.[Title], t.[Content], t.[Rating], t.[BlogId], t.[DoNotMap], 
		t_1.[UserId], t_1.[Username], t_1.[EmailAddress], t_1.[Password], t_1.[IsEnabled], t_1.[HeightInMeters] 
	from [Posts] as t 
		left join [Users] as t_1 
			on t.AuthorId = t_1.UserId 
	where ([PostId] = @l_1)",
	
	// Mapping function
	(p, u) => {
		p.Author = u;
		return p;
	},
	
	// Parameters
	new { l_1 = i },
	
	// Where to split the result set for the multi-mapping
	splitOn: "UserId").First();
```

__Using Dashing__
```
return session.Query<Post>()
	.Fetch(p => p.Author)
	.First(p => p.PostId == i);
```

In actual fact, the execution of the query, in Dashing, ends up being very similar to what happens in the Dapper version. It does a few more things in the mapping function, sure, but through [CodeDom](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/using-the-codedom) we're able to cache the generation of these methods, getting us pretty close to the same performance as the Dapper query - the extra overhead is in the parsing of the expressions and generation of the SQL.

# Successes

We've got Dashing being used in multiple production systems (that we know of) and it's been used by junior developers without much guidance. 

One of those systems, as an example, has nearly 250 tables but the configuration of those tables takes up only about 150 lines of code, which almost entirely exist of changing string lengths. As Dashing generates foreign keys and indexes for you, the performance has been high out of the box and only 40 additional indexes have had to be specifed within those 150 lines of code. The number of occasions on which the developer has had to revert to SQL is very low, only really being employed when performance is paramount. This means that, for the most part, all of the database code is easily re-factored and also easily testable as we can make use of the in-memory version of Dashing for that purpose.

# The future

I still think that Dashing is a great ORM and I've done some good work recently to improve performance and productivity: notably .Net core support, inferred inner joins and versioned entities. We'll be continuing to use it as much as possible in both existing and new systems and, as a result, have the following things planned on the Dashing roadmap:

- **Non-fetched get exceptions**  
 When you access a reference, that wasn't fetched in the query, Dashing instantiates an instance of that class using the foreign key value from the base table. Unfortunately, this sometimes ends up with developers not realising that this instance hasn't been hydrated completely. I've a wip branch for this that was never quite finished as I couldn't figure out how to support serialisation of the entities. (I've figured it out now)
- **A strongly typed SQL builder**  
 The occasions when we switch to SQL are rare. However, when we do, we lose a lot as we've switched to a string that can't be inspected or easily re-factored.  
 I'd like to solve this by building a beautiful strongly typed SQL builder. I've got another wip branch for this but it requires serious thought as I want a great API from the first version.
- **Projections**  
 Selecting only the columns you want is, at the moment, not very elegant. This is definitely something I'd like to fix but in 2 different ways. In EF you can `.Select()` your projection but this loses the original typings. I'd like to provide both that and some mechanism for re-using the existing types so that change-tracking could still work.
- **Event Handling**  
 We've got some basic event handling built in to Dashing but it needs some consolidation and re-work to bring it up to scratch.
- **Caching/Full-Text Search/Concurrency**  
 It's possible to do all of these things already but, my hope would be that, by building a great events system through Dashing we could provide simple plugins for common scenarios that you may expect to have "out of the box" in an ORM.







