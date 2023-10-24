# PostgreSQL services

How to use PostgreSQL as cache and message queue services.

## Least Recently Used Cache

Why use a separate caching applcation, when you can just use some simple functionality using PostgreSQL.

Start by creating the cache table

```sql
create table lru_cache (
  _key text not null,
  _value text not null,
  inserted_at timestamp not null default now()
);
```

Then create an index on that table hashing the `_key`.

```sql
create index lru_cache_key on lru_cache using hash(_key);
```

Now the last step is to make sure that we expire the cached entries. Simply create a `cron` job that runs every night (or anytime you want) that expires (deletes) the rows in the table older than a day or so:

```sql
delete from lru_cache where inserted_at < now() - interval '1 day';
```

Also you should tell PostgreSQL to increase its cache size by setting `shared_buffers = 10GB` in the `postgresql.conf` file. This helps with not wearing out the drive too early.

## Message Queues

You can use PostgreSQL as a simple message broker by create one single table.

Firstly, make sure you have the UUID extension activated:

```sql
create extension if not exists "uuid-ossp";
```

Now we can create the table, having a unique ID, inserted timestamp and the payload.

```sql
create table queue_table (
  id uuid default gen_random_uuid() primary key,
  inserted_at timestamp not null default now(),
  payload text not null
);
```

Then we will index the inserted messages in the ascending order (first in first out principle).

```sql
create index inserted_at_idx on queue_table (inserted_at asc);
```

Now we're ready to produce and consume.

### Enqueuing Messages

To produce a message, a simple `insert` query is the way to go.

```sql
insert into queue_table (payload) values ('Hello world!');
```

### Dequeuing Messages

To consume a message, a bit more complex `delete` query is needed. We need to make sure that we're consuming a message that is already being consumed by someone. By adding a `where` clause where the inner query will pick from the top of the queue, but by adding the `for update skip locked` we will automatically skip the row currently in transaction and go to the next row.

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
