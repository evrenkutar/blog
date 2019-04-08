---
layout: page
title: Solving Index Problem
permalink: /solving-index-problem-postgresql/
---

We have one main model where we store photos that we collect from various platforms for our clients.
We return paginated results from our backend to be efficient. To do this we use indexes on our postgresql database.

Recently we faced a slow performance for one of our campaigns. The campaign has around 16K photos. It can not be said that it is a big amount.  
Nevertheless, it was slow in response. On the other hand, there is a campaign with 390K photos but it is fast. Our db query is very simple there;   
`select * from photos WHERE campaign_id=<campaign_id> ORDER BY uploaded_at DESC;`.  

When we tried to see what happens on postgres using   
`EXPLAIN ANALYZE select * from photos WHERE campaign_id=<campaign_id> ORDER BY uploaded_at DESC;` we had the these results;

*for slow campaign (16K photos):*

```
QUERY PLAN

--------------------------------------------------------------------------------
 Limit  (cost=0.43..1989.75 rows=25 width=12) (actual time=1498.054..102721.233 rows=25 loops=1)
   ->  Index Scan Backward using photo_uploaded_at on photos (cost=0.43..1574024.20 rows=19781 width=12) (actual time=1498.052..102721.225 rows=25 loops=1)
         Filter: (category_id = 149)
         Rows Removed by Filter: 5396901
 Planning time: 0.313 ms
 Execution time: 102721.266 ms
 (6 rows)
```

*and the big (390K photos) but the fast one:*

```
QUERY PLAN

--------------------------------------------------------------------------------
 Limit  (cost=0.43..98.20 rows=25 width=12) (actual time=2.352..66.326 rows=25 loops=1)
   ->  Index Scan Backward using photo_uploaded_at on photos (cost=0.43..1574024.20 rows=402483 width=12) (actual time=2.350..66.305 rows=25 loops=1)
         Filter: (category_id = 150)
         Rows Removed by Filter: 9251
 Planning time: 3.734 ms
 Execution time: 66.383 ms
 (6 rows)
```

As it can be seen for both queries the query planner of the postgresql uses `photo_uploaded_at` index. This is because we order this db lookup by `uploaded_at` field. All looks ok.  
Except in the slow campaign we see `Rows Removed by Filter: 5396901`.  
While for the fast one it is `Rows Removed by Filter: 9251`. This is a huge difference.  
And this was the index the query use: `"photo_uploaded_at" btree (uploaded_at)`. This single column index should work in theory. But in this particular case, it doesn't work efficiently.


Since we filter by `campaign_id` we changed the index to multicolumn index as `"photo_uploaded_at" btree (campaign_id, uploaded_at)`. And the results are way better. Even the fast one above it improved \~5 times faster:

```
QUERY PLAN

--------------------------------------------------------------------------------
 Limit  (cost=0.43..28.30 rows=25 width=12) (actual time=1.464..17.437 rows=25 loops=1)
   ->  Index Scan Backward using photo_uploaded_at on photos (cost=0.43..18291.47 rows=16407 width=12) (actual time=1.462..17.422 rows=25 loops=1)
         Index Cond: (category_id = 149)
 Planning time: 0.314 ms
 Execution time: 17.473 ms
(5 rows)
```

```
QUERY PLAN

--------------------------------------------------------------------------------
 Limit  (cost=0.43..24.95 rows=25 width=12) (actual time=2.952..14.989 rows=25 loops=1)
   ->  Index Scan Backward using photo_uploaded_at on photos (cost=0.43..378948.58 rows=386399 width=12) (actual time=2.950..14.977 rows=25 loops=1)
         Index Cond: (category_id = 150)
 Planning time: 3.955 ms
 Execution time: 15.086 ms
(5 rows)
```
