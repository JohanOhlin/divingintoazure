---
layout: post
title: "Group concatenation in T-SQL using CTE"
date: 2013-11-15
tags: t-sql cte
---

<p class="intro"><span class="dropcap">I</span>n Oracle you have <a href="http://www.oracle-base.com/articles/misc/string-aggregation-techniques.php" target="_blank">listagg()</a> and in MySQL you have <a href="http://dev.mysql.com/doc/refman/5.0/en/group-by-functions.html#function_group-concat" target="_blank">group_concat()</a> to create string aggregations. But what do you have in T-SQL? The simple answer is nothing, yet, and we’re all hoping for a change in the future. But until then it’s up to us to do it by hand.</p>

In a [previous post]({{ site.baseurl }}{% post_url 2013-11-15-recursive-cte-calls-in-tsql %}) I showed how to nest up hierarchical trees with parent-child relationships. In this post I’m going to continue on that one and create a list of the managers each person has.

The previous post used a recursive CTE to create this result:

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

This result can be useful for further processing but it might not be that visually clear right away. To aggregate the manager names and show them in a one-line-list next to the user would be much more intuitive. It’s actually not that difficult and with just a little change in the query we can get an answer like this instead:

{% highlight sql linenos %}
userId      userName       managerNames         numManagers
----------- -------------- -------------------- -----------
1           John                                0
7           Claire         Vince Lynn John      3
6           Vince          Lynn John            2
5           Lynn           John                 1
4           Neil           Lynn John            2
3           Nicolas        Charles John         2
2           Charles        John                 1

(7 row(s) affected)
{% endhighlight %}

What we need to do is concatenate the manager’s names in the recursive loop (results in 18 rows with all combinations in the hierarchy) and then filter out the lines where we’ve reached the top manager and thus completed the list of managerNames. This last part is done by finally filtering by setting WHERE managerId IS WHERE (on line 28 in the query below).

One other tricky thing with aggregating the manager’s names, is that the length of the string in the anchor section and the recursive section has to be the same. If the anchor has an nvarchar(50) then the total length of the new concatenation in the recursive part has to be 50 as well. Some serious casting is needed here to make it work but it’s not that complex.

{% highlight sql linenos %}
WITH UserCTE AS (
  SELECT 
    userId as baseUserId, 
    userName as baseUserName, 
    userId, 
    userName, 
    managerId, 
    CAST('' AS NVARCHAR(50)) as managerNames, 
    0 AS numManagers
  FROM dbo.Users
    
  UNION ALL
  
  SELECT 
    usr.baseUserId,
    usr.baseUserName, 
    mgr.userId, 
    mgr.userName, 
    mgr.managerId, 
    LTRIM(CAST(usr.managerNames AS NVARCHAR(14)) + N' ' + 
         CAST(mgr.userName AS NVARCHAR(35))) AS managerNames, 
    usr.numManagers +1 AS numManagers
  FROM UserCTE AS usr
    INNER JOIN dbo.Users AS mgr
      ON usr.managerId = mgr.userId
)
SELECT baseUserId AS userId, baseUserName AS userName, 
       managerNames, numManagers 
  FROM UserCTE AS u 
  WHERE managerId IS NULL;
{% endhighlight %}

Note line 20 in the code where the length of the strings (14 + 1 + 35) equals the same length as the string in the anchor (50) on line 8.

One final thing we need to change from the original code is that we need to bring the user’s name from the anchor section because that’s the user we’re tracking through the hierarchy. That is being done by the baseUserName that is just passed on in the recursive section.