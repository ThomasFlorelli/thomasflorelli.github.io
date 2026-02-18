# What happens when PostgreSQL stops vacuuming?

You can spend a lifetime storing and accessing things in PgSQL databases without thinking too much about why and how pg vacuums dead tuples for you — the obsolete row versions left behind after every UPDATE or DELETE, which PostgreSQL keeps around rather than erasing immediately in order to support concurrent transactions (MVCC).
At least that's what I did until a weird growing issue happened at Seyna. For weeks I had been hearing team members saying that the more we delivered features, the slower the application became, at a worrying rate. We tried to identify and fix some of the slowest queries that we knew were not optimal but didn't think they would start biting so fast. The complaints were still there and waiting times kept growing until Sentry signaled a simple count query taking up to 2 minutes on our largest database. By using an EXPLAIN query I made sure that the query was using the intended indexes so it came down to a simple question: why is the index not helping?

I found this article that explains everything there is to know on the topic and confirmed in my head that this should be our main concern: https://rogerwelin.github.io/2026/02/11/postgresql-bloat-is-a-feature-not-a-bug/

I now knew that something was wrong with our dead tuples tricking pg into scanning dozens of millions of irrelevant keys.

### A few quick checks on the settings told me that the autovacuum was activated:

```sql
SELECT name, setting, source
FROM pg_settings
WHERE name IN (
  'autovacuum',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_vacuum_threshold',
  'autovacuum_vacuum_cost_delay',
  'autovacuum_max_workers'
);
```

| name                           | setting | source             |
| ------------------------------ | ------- | ------------------ |
| autovacuum                     | on      | default            |
| autovacuum_max_workers         | 3       | configuration file |
| autovacuum_vacuum_cost_delay   | 5       | configuration file |
| autovacuum_vacuum_scale_factor | 0.1     | configuration file |
| autovacuum_vacuum_threshold    | 50      | default            |

### But did not run for the past 2 months:

```sql
select
  relname,
  last_autovacuum,
  ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0), 2) AS dead_ratio,
  n_dead_tup
FROM pg_stat_user_tables
order by dead_ratio desc
```

| relname    | last_autovacuum              | dead_ratio | n_dead_tup  |
| ---------- | ---------------------------- | ---------- | ----------- |
| table_name | 2025-12-26 04:12:19.178+0100 | 128.35     | 123,705,308 |

In a healthy database, this ratio should be more or less around your autovacuum_threshold settings. We were way past that.

Two things can block the autovacuum process: stuck replication slots and open transactions.

### Replication slots wasn't our culprit, yet here is a query that was used to verify

```sql
SELECT
  slot_name,
  slot_type,
  active,
  age(xmin) AS xmin_age,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;
```

| slot_name | slot_type | active | xmin_age | lag |
| --------- | --------- | ------ | -------- | --- |

Whole bunch of emptiness here.

### But after checking running transactions, there was very little doubt left

```sql
SELECT
  pid,
  usename,
  age(clock_timestamp(), xact_start) AS transaction_age,
  age(clock_timestamp(), query_start) AS query_age,
  state,
  left(query, 100) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start ASC;
```

| pid   | usename | transaction_age               | query_age | state  | query                                |
| ----- | ------- | ----------------------------- | --------- | ------ | ------------------------------------ |
| 42385 | seyna   | 2 mons 4 days 20:07:53.308149 |           | active | a simple commented out _count_ query |

The only thing left to do was to kill the transaction and watch our CPU burn while all the autovacuum processes start running concurrently on table files way larger than they should be. An hour later everything was smooth again.

Of course a stuck transaction can bring several unwanted side effects on top of this — a held connection slot, lock contention on any rows it touched, and WAL retention preventing disk cleanup — so I urge anyone having a doubt to check and why not monitor this query for long-lasting transactions.

## Would a proper idle_in_transaction_session_timeout setting prevent this issue?

[`idle_in_transaction_session_timeout`](https://postgresqlco.nf/doc/en/param/idle_in_transaction_session_timeout/) terminates sessions that have been idle within an open transaction for too long. In this specific case, the transaction was in `active` state — a query was still running — so this setting had no effect, even though we had it configured to 1h.

The combination of [`statement_timeout`](https://postgresqlco.nf/doc/en/param/statement_timeout/) and `idle_in_transaction_session_timeout` would have handled it: `statement_timeout` kills the running query, which transitions the transaction to `idle in transaction (aborted)` — at which point `idle_in_transaction_session_timeout` kicks in and terminates the connection.
