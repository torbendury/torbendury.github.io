---
layout: post
title: Handling Evil User Input
categories: [Development, Security]
---

A small post with thoughts on handling user inputs and the when's and how's of sanitizing.

## General Thoughts on User Input

When running interactive web applications, there are multiple ways of user input involved - and thus multiple possibilities of your servers being cracked open like nuts. The processing of user input starts in your head, because every potential possibility for a user to interact with your application carries a certain risk. This starts with simple forms and search functions and goes far beyond sub-requests sent in the background that contain user data.

If you take away just one message from this post, remember it: **User input is evil.**

In a way, Murphy's Law applies here - **any user input you can think of will make it into your input field if users are given the time and opportunity**.

But let us have a look on which user input can possibly processed and saved.

## Handling XSS / Nested HTML injections

Cross-Site-Scripting might be as old as the HTML standard itself, although the first version of it is some years older than me. The good thing is that solutions for this problems are (afaik) as old as the problems themselves.

### Example

Here is a MVP example of a form that takes the first and last name of a user, taken from [w3schools.com](https://www.w3schools.com/html/html_forms.asp).

```html
<form>
  <label for="fname">First name:</label><br>
  <input type="text" id="fname" name="fname"><br>
  <label for="lname">Last name:</label><br>
  <input type="text" id="lname" name="lname">
</form>
```

When receiving user data, you will probably process it in a backend application written in languages as PHP, Java, JavaScript or Python. Since all of those languages have their own ways of implementing security measurements, I will keep it general.

### Handling

When you search the internet for preventing XSS, you often read about people that are not fans of filtering everything (or anything at all). Instead, they simply apply [HTML entity encoding](https://en.wikipedia.org/wiki/Character_encodings_in_HTML). This **automatically** turns potentially funky characters like `<` into `&lt;`.

This might be a *simple* solution - assuming you **really** do not want to accept any HTML input. In forums or comment areas, it is still common to accept (a certain subset of) HTML tags to format text. Outside of that bubble, you probably do not want users to enter valid HTML that will be rendered. **Note** that there are *some* permutations using alternative encodings that will almost everytime let something through, except you are doing very restrictive character whitelisting (regex `a-zA-Z0-9` for example) - but this will not spark joy for users.

A better way of presenting user data in your web application frontend is **templating**. Templating engines like **Jinja** come OOTB with some decent user input handling. Again, this post is not language-specific, so you need to dig into those keywords for your langauge.

Jeff Atwood also once described [how StackOverflow.com sanitizes user input](https://blog.stackoverflow.com/2008/06/safe-html-and-xss/) (in non-language-specific terms). You might think that the post is a bit old, but the problem of XSS/SQLi also is and some of the best solutions also are.

## SQL Injections

If you do not yet know what SQL injections are: These are ways of altering your SQL query statements in a away that let users concat their own query and do bad things to your database.

### Example

A simple example is user data retrieved from a dictionary that contains request data from a `POST` request sent to a backend server. It is then loaded into a SQL statement which will then be sent to the database as a query.

```python
text_user_id = request["user_id"];

text_sql = "SELECT * FROM Users WHERE UserId = " + text_user_id

db_result = db_connection.execute(text_sql)
```

This is simple and quickly done. It works because every database client engine I know still allows raw query statements which might be one of the worst practices you can implement today using a modern database client engine.

If you look at the above example and ask yourself 'What is bad about this?', I'll dig into it for you.

Let us assume that a user Bob has logged into your website, you want to present him some information about his account, so you retrieve his user from the database by using the primary key `UserId` which is `42`. Your resulting SQL statement is `SELECT * FROM Users WHERE UserId = 42` which will correctly be executed. All fine?

But now Bob has explored the network tab in the developers' tools in his favorite web browser. He sees the request made to your application and thinks 'What will happen if I...?', and it happens. Your resulting SQL statements is `SELECT * FROM Users WHERE UserId = 42 OR 1=1` which he managed to enter because your request can be reforged and resent with `curl`.

There are numerous examples for this, just use your fantasy like they do. What keeps them from using a username `42 OR 1=1`? Probably nothing, but that doesn't mean you have to actually validate the input. To the rescue, parameterized queries!

### Parameterized queries

Remember the one sentence you had to remember, if nothing else from this post? Here comes the second one.

1. User input is evil.
2. Never use raw SQL statements.

Let us quickly dive into this. Above, we saw that just concatenating incoming parameters onto a pre-built query string can (and will) break everything.

The best way to protect against this is not filtering or manually sanitizing, but rather **a religious usage of parameterized queries**. Did I forget to mention that you should **never concatenate user input** to **anything**?

When using DB engines, you will come across parameterized queries really quick. Regardless of the language you are using, they'll look similar to this:

```python
cursor = connection.cursor(prepared=True)

# Parameterized query
sql_insert_query = """ INSERT INTO Employee
                       (id, Name, Joining_date, salary) VALUES (%s,%s,%s,%s)"""

user_input = ("user_id", "Torben", "2022-10-23", 1337)

cursor.execute(sql_insert_query, user_input)

connection.commit()
```

With usage of parameterized queries, no more sanitation will be needed as the input is sanitazed and escaped by the DB engine.

### ORM

ORM (object-relational mapping) is an even easier and more readable way of handling DB queries, especially when handling user input.

In pseudo-code, it looks like the following:

```text
query_result = db_connection.query(options=options).select(User).filter(filter=filter).groupby(grouping)
```

Where your DB engine will choose the query to build from the `User` class you have defined and also the `filter` and `grouping` that can safely be entered by users.

## Conclusion

Handling user data is not easy, not at all. While you have limited time to finish your project or API, users normally do not have those limitations and/or can use automated scripts to test for XSS/SQLi vulnerabilities.

Although you can make those attack scenarios more time consuming (e.g. though rate limiting), they will eventually break through.

This is why you should definitely handle user input in the safest ways possible, if you have the resources for it, even combine several techniques to enhance security.

Above, you saw several possible ways of handling user inputs and how to safely use database connections and presenting user data.

Although the time for your project might be limited and you definitely have limited resources compared to hackers, take your time, *go and see* which possibilities you have to secure your application. Doing nothing is the worst you can do.
