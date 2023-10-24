# PostgreSQL services

How to use PostgreSQL as cache server and message queue

## LRU Cache

Create new table:

```sql
create table lru_table (
  _key text not null,
  _value text not null,
  inserted_at timestamp not null default now()
);
```

Create an index on that table:

```sql
create index lru_table_key on lru_table using hash(_key);
```

Create a cron schedule to delete rows older than a day or so to keep the database at a minimum.

Also tell PostgreSQL to increase its cache and set `shared_buffers = 10GB` in its `postgresql.conf` file.

## Message Queues

Activate UUID extensions:

```sql
create extension if not exists "uuid-ossp";
```

Create table:

```sql
create table queue_table (
  id uuid default gen_random_uuid() primary key,
  inserted_at timestamp not null default now(),
  payload text not null
);
```

Create index on inserted messages in ascending order (first in first out principle).

```sql
create index inserted_at_idx on queue_table (inserted_at asc);
```

## Enqueuing messages

To insert messages, just write a simple SQL query:

```sql
insert into queue_table (payload) values ('hello');
```

## Dequeuing messages

To consume a message, a simple `DELETE` will do it:

```sql
delete
from queue_table qt
where qt.id =
  (select qt_inner.id
   from queue_table qt_inner
   order by qt_inner.inserted_at asc
     for update skip locked
   limit 1)
returning qt.id, qt.inserted_at, qt.payload;
```
