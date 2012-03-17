---
layout: post
title: "Random Per Row"
date: 2012-03-08 21:09
comments: false
published: false
categories: [SQL, Code]
---

I needed a way to randomize the timestamps on some records that I would be inserting via a SQL script, in order to make them look realistic.  To generate the random times of day which would be appended to the dates, I knew that I should start by generating random numbers.  The function of choice is T-SQL is `RAND()`:

``` sql
SELECT RAND();

Result:
0.719425963
```

Like most random number generators, it generates a floating point number between 0 and 1, so I needed to scale the result to the 12 hour range across which a typical business day spans, through multiplication:

```
SELECT RAND() * 12 AS random_hour;

Result:
5.12519698
```

While I could convert the fractional component of the number to a minute within the hour, I am more comfortable generating the minute separately because it will be more easily understood. I will eliminate the fractional component using `FLOOR()`:

```
SELECT FLOOR(RAND() * 12) AS random_hour;

Result:
8
```

Next I needed to add an offset to produce an hour that was within the average workday, 7 AM to 7 PM:

```
SELECT FLOOR(RAND() * 12) + 7 AS random_hour;
```

``` sql
SELECT
    DATEADD(minute, FLOOR(RAND() * 60),
    DATEADD(hour, FLOOR(RAND() * 12) + 7,
    r.last_updated_date)) AS last_updated_date
FROM records r

Result:
| last_updated_date   |
-----------------------
| 2012-03-01 09:27:00 |
| 2012-03-02 09:27:00 |
| 2012-03-03 09:27:00 |
| 2012-03-04 09:27:00 |
```

Unfortunately, `RAND()` generates a pseudo-random number only once per query, not per row.  But we can induce a random-per-row effect by giving the function a random seed that changes on each row.  We'll use NEWID() to create a unique identifier, convert it to a binary number, and pass it to RAND().  

``` sql
SELECT
    DATEADD(minute, FLOOR(RAND(CONVERT(BINARY, NEWID())) * 60),
    DATEADD(hour, FLOOR(RAND(CONVERT(BINARY, NEWID())) * 12) + 7,
    r.last_updated_date)) AS last_updated_date
FROM records r

Result:
| last_updated_date   |
-----------------------
| 2012-03-01 09:27:00 |
| 2012-03-02 13:13:00 |
| 2012-03-03 12:02:00 |
| 2012-03-04 13:55:00 |
```

