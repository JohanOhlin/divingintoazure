---
layout: post
title: "How to use recursive CTE calls in T-SQL"
date: 2013-11-15
tags: t-sql cte
---

<p class="intro"><span class="dropcap">A</span> common table expression (CTE) in T-SQL is used by many to get around the internal referencing problem in derived tables. It also gives you a better overview of your query compared to having nested derived tables. But one really nice feature with CTEs is that you can let it call itself recursively and nest up trees of parent-child relationships.</p>

When making a recursive CTE you divide it into two sections joined by a UNION ALL where the first section is the anchor, and is only called once, while the second section do the recursion and is called repeatedly until it returns an empty result.

The table used in the examples below is very simple with only three columns: userId, userName and managerId.

A simple CTE using this table could look like this:

{% highlight sql %}
WITH UserCTE AS (
  SELECT userId, userName, managerId
  FROM dbo.Users
)
SELECT * FROM UserCTE;
{% endhighlight %}

This query will give us all the users in the list and their closest manager, if they have one (top manager doesn’t have a manager).

{% highlight sql %}
userId      userName         managerId
----------- ---------------- -----------
1           John             NULL
2           Charles          1
3           Nicolas          2
4           Neil             5
5           Lynn             1
6           Vince            5
7           Claire           6

(7 row(s) affected)
{% endhighlight %}

The problem with this list is that you have to search yourself to find out where in the hierarchy a person is and more processing is needed before data can be displayed. This is where the recursive calls come in handy.

{% highlight sql %}
WITH UserCTE AS (
  SELECT userId, userName, managerId,0 AS steps
  FROM dbo.Users
  WHERE userId = 7
    
  UNION ALL
  
  SELECT mgr.userId, mgr.userName, mgr.managerId, usr.steps +1 AS steps
  FROM UserCTE AS usr
    INNER JOIN dbo.Users AS mgr
      ON usr.managerId = mgr.userId
)
SELECT * FROM UserCTE AS u;
{% endhighlight %}

In the example above you can see the recursive part on the highlighted rows. A step counter is used to calculate degrees of separation from the user to the managers. From just a few people in a hierarchy you can get plenty of relationships, so we also added a restriction to only show user with id 7.

The result of this query would be:

{% highlight sql %}
userId      userName         managerId   steps
----------- ---------------- ----------- -----------
7           Claire           6           0
6           Vince            5           1
5           Lynn             1           2
1           John             NULL        3

(4 row(s) affected)
{% endhighlight %}

What about showing more than one user?

If we were to remove the limitation of only showing user with id 7, the result would be a bit chaotic and unordered.

{% highlight sql %}
userId      userName         managerId   steps
----------- ---------------- ----------- -----------
1           John             NULL        0
2           Charles          1           0
3           Nicolas          2           0
4           Neil             5           0
5           Lynn             1           0
6           Vince            5           0
7           Claire           6           0
6           Vince            5           1
5           Lynn             1           2
1           John             NULL        3
5           Lynn             1           1
1           John             NULL        2
1           John             NULL        1
5           Lynn             1           1
1           John             NULL        2
2           Charles          1           1
1           John             NULL        2
1           John             NULL        1

(18 row(s) affected)
{% endhighlight %}

The reason for this is that we first list all users and then add the recursive connections one by one at the end of the list. T-SQL is based upon SET theory and as such is unordered by definition. In fact, unless you use ORDER BY, there is no guarantee what so ever that the result from running the same query twice will generate rows in the same order. Thus we need to find a way to order the rows, user by user as we track them up the hierarchy tree.

One way of doing it is to save the user id from the anchor section in the CTE and pass it along in the recursive call. In the result from the CTE you can then sort by this anchor user id.

{% highlight sql %}
WITH UserCTE AS (
  SELECT userId as baseUserId, userId, userName, managerId, 0 AS steps
  FROM dbo.Users
    
  UNION ALL
  
  SELECT usr.baseUserId, mgr.userId, mgr.userName, 
    mgr.managerId, usr.steps +1 AS steps
  FROM UserCTE AS usr
    INNER JOIN dbo.Users AS mgr
      ON usr.managerId = mgr.userId
)
SELECT * 
  FROM UserCTE AS u 
  ORDER BY baseUserId, steps;
{% endhighlight %}

The result of the query would then be a sorted list where the manager trail is in order for each and every one of the users.

{% highlight sql %}
baseUserId  userId      userName         managerId   steps
----------- ----------- ---------------- ----------- -----------
1           1           John             NULL        0
2           2           Charles          1           0
2           1           John             NULL        1
3           3           Nicolas          2           0
3           2           Charles          1           1
3           1           John             NULL        2
4           4           Neil             5           0
4           5           Lynn             1           1
4           1           John             NULL        2
5           5           Lynn             1           0
5           1           John             NULL        1
6           6           Vince            5           0
6           5           Lynn             1           1
6           1           John             NULL        2
7           7           Claire           6           0
7           6           Vince            5           1
7           5           Lynn             1           2
7           1           John             NULL        3

(18 row(s) affected)
{% endhighlight %}

As you can see in the result the baseUserId contains the id of the user we’re tracking and we can follow that user all the way up to John, the top manager.

#### Conclusions
When you have smaller hierarchical sets of data, CTEs can be handy to produce nice sets of data. But the recursive element of this processing can be heavy for larger tables with deep hierarchies and should thus be used carefully. But in smaller sets, or when later processing is difficult, you’ve got a key tool in the CTE’s ability to call itself.