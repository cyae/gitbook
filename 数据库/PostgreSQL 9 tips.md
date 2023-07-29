The common thread linking most of these gotchas is scalability. They're things that won't affect you while your database is small. But if one day you want your database not to be small, it pays to think about them in advance.

It's an easy trap to fall into if you're not aware of it, because all your queries in local development will typically run perfectly. And probably in production too at first, you'll have no issues. But as your application grows, the volume of data and complexity of your queries both increase. It's only then that you'll start to encounter problems, a textbook _"but it worked on my machine"_ scenario.

## PLEASE DO NOT:

## 1. Keep the default value for `work_mem`
This setting governs how much memory is available to each query operation before it must start writing data to temporary files on disk. 
When `work_mem` becomes over-utilised, you'll see latency spikes as data is paged in and out, causing hash table and sorting operations to run much slower.
A good value depends on multiple factors: 
- the size of your Postgres instance, 
- the frequency and complexity of your queries, 
- the number of concurrent connections.

Running your logs through [pgbadger](https://pgbadger.darold.net/) is one way to look for warning signs. Another way is to use an automated 3rd-party system that alerts you before it becomes an issue, such as [pganalyze](https://pganalyze.com/)

In prod, we often starts with:
$work\_mem = \frac{\$INSTANCE\_MEMORY * 0.8 - shared\_buffers} {\#ACTIVE\_CONNECTIONS}$
you can also override `work_mem` on a per-transaction basis using `SET LOCAL work_mem`

## 2. Push all your application logic into Postgres functions and procedures

Pretty obvious. This is essentially App layer v.s. Storage Layer.

## 3. Use lots of triggers

Firstly, they're less efficient than some of the alternatives. Requirements that can be implemented using [generated columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html) or [materialized views](https://www.postgresql.org/docs/current/rules-materializedviews.html) should use those abstractions, as they're better optimised by Postgres internally.

Secondly, there's a hidden gotcha lurking in how triggers tend to encourage event-oriented thinking. **As you know, it's good practice in SQL to batch related `INSERT` or `UPDATE` queries together, so that you lock a table once and write all the data in one shot.** You probably do this in your application code automatically, without even needing to think about it. But triggers can be a blindspot.

A useful discipline here is to restrict yourself to a single `BEFORE` trigger and a single `AFTER` trigger on each table. Give your trigger functions generic names like `before_foo` and `after_foo`, then keep all the logic inline inside one function. Use [`TG_OP`](https://www.postgresql.org/docs/current/plpgsql-trigger.html#PLPGSQL-DML-TRIGGER) to distinguish the trigger operation. If the function gets long, break it up with some comments but don't be tempted to refactor to smaller functions. This way it's easier to ensure writes are implemented efficiently, plus it also limits the overhead of managing an extended call stack in Postgres.

## 4. Use `NOTIFY` heavily
Using [`NOTIFY`](https://www.postgresql.org/docs/current/sql-notify.html), you can extend the reach of triggers into your application layer. That's handy if you don't have the time or the inclination to manage a dedicated message queue, but once again it's not a cost-free abstraction.

If you're generating lots of events, the resources spent on notifying listeners will not be available elsewhere. This problem can be exacerbated if your listeners need to read further data to handle event payloads. Then you're paying for every `NOTIFY` event plus every consequential read in the handler logic. Just as with triggers, this can be a blindspot that hides opportunities to batch those reads together and reduce load on your database.

Instead of `NOTIFY`, consider writing events to a FIFO table and then consume them in batches at regular cadence. The right cadence depends on your application, maybe it's a few seconds or perhaps you can get away with a few minutes. Either way it will reduce the load, leaving more CPU and memory available for other things.

A possible schema for your event queue table might look like this:

```SQL
CREATE TABLE event_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type text NOT NULL,
  data jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  occurred_at timestamptz NOT NULL,
  acquired_at timestamptz,
  failed_at timestamptz
);
```

With that in place you could acquire events from the queue like so:

```SQL
UPDATE event_queue
SET acquired_at = now()
WHERE id IN (
  SELECT id
  FROM event_queue
  WHERE acquired_at IS NULL
  ORDER BY occurred_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1000 -- Set this limit according to your usage
)
RETURNING *;
```

Setting `acquired_at` on read and using `FOR UPDATE SKIP LOCKED` guarantees each event is handled only once. After they've been handled, you can then delete the acquired events in batches too (there are better options than Postgres for permanently storing historical event data of unbounded size).

## 5. Don't use `EXPLAIN ANALYZE` on real data

[`EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html) is a core tool in every backend engineer's kit. I'm sure you diligently check your query plans for the dreaded `Seq Scan` already. But Postgres can return more accurate plan data if you use [`EXPLAIN ANALYZE`](https://www.postgresql.org/docs/current/using-explain.html#USING-EXPLAIN-ANALYZE), because that **actually executes** the query. Of course, you don't want to do that in production. So to use `EXPLAIN ANALYZE` well, there are a few steps you should take first.

Any query plan is only as good as the data you run it against. There's no point running `EXPLAIN` against a local database that has a few rows in each table. Maybe you're fortunate enough to have a comprehensive seed script that populates your local instance with realistic data, but even then there's a better option.

It's really helpful to set up a dedicated sandbox instance alongside your production infrastructure, regularly restored with a recent backup from prod, specifically for the purpose of running `EXPLAIN ANALYZE` on any new queries that are in development. Make the sandbox instance smaller than your production one, so it's more constrained than prod. Now `EXPLAIN ANALYZE` can give you confidence about how your queries are expected to perform after they've been deployed. If they look good on the sandbox, there should be no surprises waiting for you when they reach production.

## 6. Prefer CTEs over subqueries

_**EDIT:**_ _Thanks to [Randommaggy](https://www.reddit.com/user/Randommaggy/) and [Ecksters](https://www.reddit.com/user/Ecksters/) for [pointing out](https://www.reddit.com/r/PostgreSQL/comments/12w574s/comment/jhdrd2n/) the subquery suggestion in the following section is outdated. Since version 12, Postgres has been much better at optimising CTEs and will often just replace the CTE with a subquery anyway. I've left the section in place as the broader point about comparing approaches with `EXPLAIN` still stands and the "CTE Materialization" docs remain a worthwhile read. But bear in mind the comment thread linked above!_

If you're regularly using `EXPLAIN` this one probably won't catch you out, but it's caught me out before so I want to mention it explicitly.

Many engineers are bottom-up thinkers and CTEs (i.e. [`WITH` queries](https://www.postgresql.org/docs/current/queries-with.html)) are a natural way to express bottom-up thinking. But they may not be the most performant way.

Instead I've found that [subqueries](https://www.postgresql.org/docs/current/functions-subquery.html) will often execute much faster. Of course it depends entirely on the specific query, so I make no sweeping generalisations other than to suggest you should `EXPLAIN` both approaches for your own complex queries.

There's a discussion of the underlying reasons in the "[CTE Materialization](https://www.postgresql.org/docs/current/queries-with.html#id-1.5.6.12.7)" section of the docs, which describes the performance tradeoffs more definitively. It's a good summary, so I won't waste your time trying to paraphrase it here. Go and read that if you want to know more.

## 7. Use recursive CTEs for time-critical queries

If your data model is a graph, your first instinct will naturally be to traverse it recursively. Postgres provides [recursive CTEs](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) for this and they work nicely, even allowing you to handle self-referential/infinitely-recursive loops gracefully. But as elegant as they are, they're not fast. And as your graph grows, performance will decline.

A useful trick here is to think about how your application traffic stacks up in terms of reads versus writes. It's common for there to be many more reads than writes and in that case, you should consider denormalising your graph to a materialized view or table that's better optimised for reading. If you can store each queryable subgraph on its own row, including all the pertinent columns needed by your queries, reading becomes a simple (and fast) `SELECT`. The cost of that is write performance of course, but it's often worth it for the payoff.

## 8. Don't add indexes to your foreign keys

Postgres doesn't automatically create indexes for foreign keys. This may come as a surprise if you're more familiar with MySQL, so pay attention to the implications as it can hurt you in a few ways.

The most obvious fallout from it is the performance of joins that use a foreign key. But those are easily spotted using `EXPLAIN`, so are unlikely to catch you out.

Less obvious perhaps is the performance of `ON DELETE` and `ON UPDATE` behaviours. If your schema relies on cascading deletes, you might find some big performance gains by adding indexes on foreign keys.

## 9. Compare indexed columns with `IS NOT DISTINCT FROM`

When you use regular [comparison operators](https://www.postgresql.org/docs/9.0/functions-comparison.html) with `NULL`, the result is also `NULL` instead of the boolean value you might expect. One way round this is to replace `<>` with `IS DISTINCT FROM` and replace `=` with `IS NOT DISTINCT FROM`. These operators treat `NULL` as a regular value and will always return booleans.

However, whereas `=` will typically cause the query planner to use an index if one is available, `IS NOT DISTINCT FROM` bypasses the index and will likely do a `Seq Scan` instead. This can be confusing the first time you notice it in the output from `EXPLAIN`.

If that happens and you want to force a query to use the index instead, you can make the null check explicit and then use `=` for the not-null case.

In other words, if you have a query that looks like this:

```sql
SELECT * FROM foo
WHERE bar IS NOT DISTINCT FROM baz;
```

You can do this instead:

```sql
SELECT * FROM foo
WHERE (bar IS NULL AND baz IS NULL)
OR bar = baz;
```

## More

- Configure Vacuum and maintenance_work_mem regularly if your DB size increases, if you allocate too much or too often it can clog up your memory.

- If you plan on deleting more than a 10,000 rows regularly, maybe you should look at partition, it's surprisingly very slow to delete that "much" data. And even more with foreign key.

- Index on Boolean is useless, it's an easy mistake that will take memory and space disk for nothing. 
    - However if the field is highly biased (e.g. 90 or 99% one value) it can be useful to create a partial index on the rarer value. 
    - Even better is to create a partial index on the other stuff filtered _by_ that value, especially if the smaller set is the commonly queried one (e.g. soft-deletes).

- Broad indices are easier to maintain but if you can have multiple smaller indices with WHERE condition it will be much faster

- You can speed up, by a huge margin, big string indices with md5/hash index (only relevant for exact match)

- Postgres as a queue is definitely working and scales pretty far, see [details](https://www.crunchydata.com/blog/message-queuing-using-native-postgresql)

- Related: be sure to understand the difference between transaction vs explicit locking, a lot of people assume too much from transaction and it will eventually breaks in prod.
