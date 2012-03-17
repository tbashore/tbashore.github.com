---
layout: post
title: "Find Gaps in Data"
date: 2012-03-17 08:33
comments: false 
categories: [SQL, Code]
---

I recently had a project where data was reported missing.  The records in the system were replicated according to a schedule (e. g. quarterly, monthly), but some of the instances had gone missing (possibly by a bug in an automated cleanup process) and I was tasked with filling in the gaps.  If I was to fill in the gaps I would need to find the gaps first.  Yet there were many thousands of records in the table, so a manual search was out of the question.

``` sql Example of quarterly records with missing instances

| due_date   |
--------------
| 2011-03-07 |
| 2011-06-06 |
| 2011-12-05 |
| 2012-03-05 |
| 2012-09-03 |

```

In the example above, you'll notice a pattern where each date is roughly three months from the previous, with the exception of two dates that have been skipped: September 2011 and June 2012.  Clearly I needed some way to recognize such a pattern with a query.

After a few moments of thought I realized that I usually use an outer join to find missing data (nulls).  What I needed was a list of dates that probably had existed, so that I could join it with the list of existing dates to find the gaps.  I started by creating a [tally table][tt] of increments that I would need to generate a periodic list of dates:

``` sql Tally table

DECLARE @inc TABLE (
	i INT
);
WITH Inc AS (
	SELECT 0 AS i
	UNION ALL
	SELECT i + 1 AS i FROM Inc WHERE i < 12
)
INSERT INTO @inc
SELECT i FROM Inc

SELECT i FROM @inc

| i  |
------
| 0  |
| 1  |
| 2  |
| 3  |
| 4  |
| 5  |
| 6  |
| 7  |
| 8  |
| 9  |
| 10 |
| 11 |
| 12 |

```

Then I took the most recent instance of an existing record and subtracted each of the increments from it to get the probable history of dates:

``` sql Probable dates

SELECT
    r.task_name,
    MAX(r.due_date) AS max_due_date,
    DATEADD(month, -3 * i.i, MAX(r.due_date)) AS probable_due_date
FROM
    records r,
    @inc i
GROUP BY
    r.task_name,
    i.i
HAVING DATEADD(month, -3 * i.i, MAX(r.due_date)) >= '2011-01-01'

| task_name | max_due_date | probable_due_date |
------------------------------------------------
| Task 1    | 2012-09-03   | 2012-09-03        | 
| Task 1    | 2012-09-03   | 2012-06-03        | 
| Task 1    | 2012-09-03   | 2012-03-03        | 
| Task 1    | 2012-09-03   | 2011-12-03        | 
| Task 1    | 2012-09-03   | 2011-09-03        | 
| Task 1    | 2012-09-03   | 2011-06-03        | 
| Task 1    | 2012-09-03   | 2011-03-03        | 
| Task 2    | 2012-08-10   | 2012-08-10        | 
| Task 2    | 2012-08-10   | 2012-05-10        | 
| Task 2    | 2012-08-10   | 2012-02-10        | 
| Task 2    | 2012-08-10   | 2011-11-10        | 
| Task 2    | 2012-08-10   | 2011-08-10        | 
| Task 2    | 2012-08-10   | 2011-05-10        | 

```

I can take this list of probable dates and perform an outer join with the actual dates, and any resulting row with a null due date is one that is missing.  Of course, an exact join will not work because there is some variability in the scheduling.  That is, the day of the month changes because the day of the week is fixed.  So, I will add a margin of plus or minus one week.

``` sql

SELECT
    pr.task_name,
    pr.probable_due_date,
    r.due_date
FROM (
    SELECT
        r.task_name,
        MAX(r.due_date) AS max_due_date,
        DATEADD(month, -3 * i.i, MAX(r.due_date)) AS probable_due_date
    FROM
        records r,
        @inc i
    GROUP BY
        r.task_name,
        i.i
	) pr LEFT OUTER JOIN
	records r ON
		r.due_date >= DATEADD(week, -1, pr.probable_due_date) AND
		r.due_date <= DATEADD(week, 1, pr.probable_due_date)
WHERE pr.probable_due_date >= '2011-01-01'
ORDER BY
	pr.task_name,
	pr.probable_due_date,
    r.due_date

| task_name | probable_due_date | due_date   |
----------------------------------------------
| Task 1    | 2011-03-03        | 2011-03-07 |
| Task 1    | 2011-06-03        | 2011-06-06 |
| Task 1    | 2011-09-03        | NULL       |
| Task 1    | 2011-12-03        | 2011-12-05 |
| Task 1    | 2012-03-03        | 2012-03-05 |
| Task 1    | 2012-06-03        | NULL       |
| Task 1    | 2012-09-03        | 2012-09-03 |
| Task 2    | 2011-02-10        | 2011-02-11 | 
| Task 2    | 2011-05-10        | 2011-05-13 | 
| Task 2    | 2011-08-10        | NULL       | 
| Task 2    | 2011-11-10        | NULL       |
| Task 2    | 2012-02-10        | 2012-02-10 |
| Task 2    | 2012-05-10        | 2012-05-11 |
| Task 2    | 2012-08-10        | 2012-08-10 |

```

Great! We found the gaps.  All that is left is to fill in the gaps with an insert.

``` sql

INSERT INTO records (
    task_name,
    due_date
)
SELECT
    pr.task_name,
    DATEADD(day, DATEPART(dw, pr.max_due_date) - DATEPART(dw, pr.probable_due_date), pr.probable_due_date)
FROM (
    SELECT
        r.task_name,
        MAX(r.due_date) AS max_due_date,
        DATEADD(month, -3 * i.i, MAX(r.due_date)) AS probable_due_date
    FROM
        records r,
        @inc i
    GROUP BY
        r.task_name,
        i.i
	) pr LEFT OUTER JOIN
	records r ON
		r.due_date >= DATEADD(week, -1, pr.probable_due_date) AND
		r.due_date <= DATEADD(week, 1, pr.probable_due_date)
WHERE r.due_date IS NULL

```

Here is a [SQL Fiddle][fiddle] that demonstrates the techniques from this post.

[tt]: http://www.google.com/
[fiddle]: http://www.sqlfiddle.com/#!3/eb1d0/1
