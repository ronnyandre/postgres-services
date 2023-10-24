# PostgreSQL services

Some PostgreSQL tricks that can be nifty.

## Least Recently Used Cache

Why use a separate caching applcation, when you can just use some simple functionality using PostgreSQL.

Start by creating the cache table

```sql
CREATE TABLE lru_cache (
  _key TEXT NOT NULL,
  _value TEXT NOT NULL,
  inserted_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Then create an index on that table hashing the `_key`.

```sql
CREATE INDEX lru_cache_key ON lru_cache USING HASH(_key);
```

Now the last step is to make sure that we expire the cached entries. Simply create a `cron` job that runs every night (or anytime you want) that expires (deletes) the rows in the table older than a day or so:

```sql
DELETE FROM lru_cache WHERE inserted_at < NOW() - INTERVAL '1 day';
```

Also you should tell PostgreSQL to increase its cache size by setting `shared_buffers = 10GB` in the `postgresql.conf` file. This helps with not wearing out the drive too early.

## Message Queues

You can use PostgreSQL as a simple message broker by create one single table.

Firstly, make sure you have the UUID extension activated:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

Now we can create the table, having a unique ID, inserted timestamp and the payload.

```sql
CREATE TABLE queue_table (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  inserted_at TIMESTAMP NOT NULL DEFAULT NOW(),
  payload TEXT NOT NULL
);
```

Then we will index the inserted messages in the ascending order (first in first out principle).

```sql
CREATE INDEX inserted_at_idx ON queue_table (inserted_at ASC);
```

Now we're ready to produce and consume.

### Enqueuing Messages

To produce a message, a simple `insert` query is the way to go.

```sql
INSERT INTO queue_table (payload) VALUES ('Hello world!');
```

### Dequeuing Messages

To consume a message, a bit more complex `delete` query is needed. We need to make sure that we're consuming a message that is already being consumed by someone. By adding a `where` clause where the inner query will pick from the top of the queue, but by adding the `for update skip locked` we will automatically skip the row currently in transaction and go to the next row.

```sql
DELETE
FROM queue_table qt
WHERE qt.id =
  (SELECT qt_inner.id
   FROM queue_table qt_inner
   ORDER BY qt_inner.inserted_at ASC
     FOR UPDATE SKIP LOCKED
   LIMIT 1)
RETURNING qt.id, qt.inserted_at, qt.payload;
```

## Stop Plus Addressing For Signups

So, you want to avoid people misuse your free tier service by registering `email@email.com`, `email+1@email.com`, `email+2@email.com` and so on? You can do that by adding a generated column that removes the plus and whatever's after that sign in the email address and making that generated column unique.

The table would look like this:

```sql
CREATE TABLE user_table (
	email VARCHAR(255),
	email_normal VARCHAR(255) GENERATED ALWAYS AS (
		SPLIT_PART(
			SPLIT_PART(email, '@', 1),
			'+',
			1
		) || '@' || SPLIT_PART(email, '@', -1)
	) STORED UNIQUE
);
```

Let me explain a bit. First we add the `email` column. This is the field you would normally validate against when logging in, nothing special about that.

But that next generated column `email_normal` is the "normalized" version of the email. This consists of splitting the email address into two. The part before and after the `@` sign. And then for the first part (before the `@` sign), split the text again at the `+` sign and return only the first part. Then you combine whatever was before the `+` sign and after the `@` sign, having the email normalized. The icing on the cake is to add a `unique` clause on that `email_normal` generated column, rendering any duplicate of unique emails impossible.

So, this would work perfectly fine:

```sql
INSERT INTO user_table (email) VALUES ('email+service@email.com`)
```

But in the database it will look like this:

```sql
SELECT * FROM user_table;
```

```
+-------------------------+-----------------+
|  email                  | email_normal    |
+-------------------------+-----------------+
| email+service@email.com | email@email.com |
+-------------------------+-----------------+
```

However, the following would generate an error on second insert (in either order it is inserted):

```sql
INSERT INTO user_table (email) VALUES ('email+service@email.com');
INSERT INTO user_table (email) VALUES ('email+otherservice@email.com');
INSERT INTO user_table (email) VALUES ('email@email.com');
```
