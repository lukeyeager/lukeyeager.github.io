---
layout: post
title: Creating performant indices on postgres jsonb columns
---

JSONB columns in postgres are cool. You can add arbitrary, structured data to rows without needing to update your schema.

{% highlight sql %}
-- Example of the sort of thing you can do with JSONB columns
=> CREATE TABLE players (id SERIAL PRIMARY KEY, data JSONB);
=> INSERT INTO players (data) VALUES ('{"name": "PlayerA"}');
=> UPDATE players SET data=jsonb_set(data, '{health}', '0.5') WHERE data->>'name' = 'PlayerA';
=> INSERT INTO players (data) VALUES ('{"name": "PlayerB", "health": 0.4}');
=> SELECT * FROM players ORDER BY (data->>'health')::float;
 id |                data
----+------------------------------------
  2 | {"name": "PlayerB", "health": 0.4}
  1 | {"name": "PlayerA", "health": 0.5}
(2 rows)
{% endhighlight %}

Obviously, you're going to want to index some of these JSON keys eventually. When you do, look out for unnecessary casts which can kill your query performance:

{% highlight sql %}
-- (BAD) Each column in the index is cast to a known type
=> CREATE INDEX rankings_query ON rankings ( ((data->>'competition')), ((data->>'caught_cheating')::boolean), ((data->>'ranking')::float) );
=> EXPLAIN ANALYZE SELECT * FROM rankings WHERE data->>'competition' = 'doom' AND (data->>'caught_cheating')::boolean = false ORDER BY (data->>'ranking')::float;
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Scan using rankings_query on rankings  (cost=0.14..8.17 rows=1 width=44) (actual time=0.018..0.023 rows=9 loops=1)
   Index Cond: (((data ->> 'competition'::text) = 'doom'::text) AND (((data ->> 'caught_cheating'::text))::boolean = false))
   Filter: (NOT ((data ->> 'caught_cheating'::text))::boolean) -- <-- HERE
 Planning time: 0.147 ms
 Execution time: 0.039 ms
(5 rows)

-- (GOOD) Each column in the index is left as type JSONB
=> CREATE INDEX rankings_query ON rankings ( ((data->'competition')), ((data->'caught_cheating')), ((data->'ranking')) );
=> EXPLAIN ANALYZE SELECT * FROM rankings WHERE data->'competition' = '"doom"' AND data->'caught_cheating' = 'false' ORDER BY data->'ranking';
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Index Scan using rankings_query on rankings  (cost=0.14..8.16 rows=1 width=68) (actual time=0.019..0.022 rows=9 loops=1)
   Index Cond: (((data -> 'competition'::text) = '"doom"'::jsonb) AND ((data -> 'caught_cheating'::text) = 'false'::jsonb))
 Planning time: 0.137 ms
 Execution time: 0.038 ms
(4 rows)
{% endhighlight %}

Here are two analyses on similar queries in a production database with a lot more data (note the "Execution time"):

{% highlight sql %}
-- Indexed by casting to a known type
=> explain analyze select count(*) from my_table where v->>'my_field1' = 'foo' and (v->>'my_field2')::boolean = true;
                                                                                QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=108247.06..108247.07 rows=1 width=8) (actual time=33289.910..33289.911 rows=1 loops=1)
   ->  Bitmap Heap Scan on my_table  (cost=40025.16..104347.37 rows=1559876 width=0) (actual time=1685.160..32754.429 rows=1559876 loops=1)
         Recheck Cond: ((v ->> 'my_field1'::text) = 'foo'::text)
         Filter: ((v ->> 'my_field2'::text))::boolean
         Heap Blocks: exact=29176
         ->  Bitmap Index Scan on my_index  (cost=0.00..39635.19 rows=1559876 width=0) (actual time=1679.954..1679.954 rows=1559876 loops=1)
               Index Cond: (((v ->> 'my_field1'::text) = 'foo'::text) AND (((v ->> 'my_field2'::text))::boolean = true))
 Planning time: 0.142 ms
 Execution time: 33289.971 ms
(9 rows)

-- Indexed by leaving as JSONB
=> explain analyze select count(*) from my_table where v->'my_field1' = '"foo"' and v->'my_field2' = 'true';
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=157.43..157.44 rows=1 width=8) (actual time=1329.478..1329.479 rows=1 loops=1)
   ->  Bitmap Heap Scan on my_table  (cost=4.83..157.33 rows=39 width=0) (actual time=518.394..1002.934 rows=1559876 loops=1)
         Recheck Cond: (((v -> 'my_field1'::text) = '"foo"'::jsonb) AND ((v -> 'my_field2'::text) = 'true'::jsonb))
         Heap Blocks: exact=29176
         ->  Bitmap Index Scan on my_index  (cost=0.00..4.82 rows=39 width=0) (actual time=513.360..513.360 rows=1559876 loops=1)
               Index Cond: (((v -> 'my_field1'::text) = '"foo"'::jsonb) AND ((v -> 'my_field2'::text) = 'true'::jsonb))
 Planning time: 0.218 ms
 Execution time: 1329.513 ms
(8 rows)
{% endhighlight %}
