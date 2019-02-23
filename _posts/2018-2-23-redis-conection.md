---
layout: post
title: How to prevent data loss with Redis.
---

Sometimes we could miss something small but quite powerful and useful in some projects.
For me it was "redis connection name" - which for first look seems like smth super specific and just for meta information.

Here I will try explain how redis connection name could help you to process a lot of data
in super "safe" mode.

[Redis set connection name](https://redis.io/commands/client-setname) - it's a way to specified some name/key/id to your redis connection. Plus, you can extract all clients info using  [client list](https://redis.io/commands/client-list) command.

Together these commands can be useful for analytic porpose. For example in Todoist we have redis [workers](https://en.wikipedia.org/wiki/Job_(computing\)) for background tasks. And using redis connection name we could understand amount of processes which producing and consuming jobs.

![_config.yml]({{ site.baseurl }}/images/redis_connection_1.jpg)

But I found that redis connection name could be in some way a "global lock" beetwen multiple processes in redis. I will try to explain this based on a simple task:

Image we have a sql table which we want to put in redis list (use it as a queue) and then calculate something in parallel way. The main goal is to read this table using multiple processes and make whole system super "safe".

![_config.yml]({{ site.baseurl }}/images/redis_connection_2.jpg)

What does it mean "super safe"? We should not:

- Lose any data (on a way from table to redis).
- Duplicate any data.
- Don't lose and duplicate data if something failed (even someone cut the power :) ).

I will try to explain my solution based on redis connection name:

### Step1. Split table by [batches](https://en.wikipedia.org/wiki/Batch_processing).

Let's split whole table by bathes/chunks and then iterate over table rows in batch and
push everything (in one commit) to redis list:


![_config.yml]({{ site.baseurl }}/images/redis_connection_3.jpg)


```python
for chunk_index in range(CHUNKS_COUNT):
    # generate batch "borders" based on chunk index
    l_id = CHUNK_SIZE * chunk_index
    r_id = CHUNK_SIZE * (chunk_index + 1)

    # reading batch of data from table
    mysql_cursor.execute(
        "SELECT * "
        "FROM table "
        "WHERE id >= %s and id < %s" % (l_id, r_id))

    # it's important to use pipeline here, because in case
    # of failure during reading process some tasks could be left
    # in list, some will be not processed (which in future will
    # call data loss or duplication)
    redis_pipeline = redis_client.pipeline()
    for row in mysql_cursor:
        redis_pipeline.lpush("rows", row)
    redis_pipeline.execute()
```

With "redis pipeline" seems like code could meet all requirements, BUT:

- We can't read batches in "parallel way", because processes will just read them from
1..CHUNKS_COUNT simultaneously and duplicate all data.

- What if something happened inside cursor iteration? The process had beed stopped and
data not delivered to redis. When we decide to "repeat" reading process for this batch we will not aware of what batch we should "repeat"

### Step2. Tag processed batches. 

Let's try to solve the problems above using [redis bitmap](https://redis.io/commands/SETBIT). We could set bit to 1 if batches had been processed and 0 otherwise.

```python
for chunk_index in range(CHUNKS_COUNT):
    # let's see if we already processed this chunk of data
    bit = redis_client.getbit("chunks", chunk_index)

    # if amount of bits set to 1 more then CHUNKS_COUNT 
    # seems like we processed everything
    if redis_client.bitcount("chunks") > CHUNKS_COUNT:
        break

    if bit == 1: continue

    l_id = CHUNK_SIZE * chunk_index
    r_id = CHUNK_SIZE * (chunk_index + 1)

    redis_pipeline = redis_client.pipeline()

    mysql_cursor.execute(
        "SELECT * "
        "FROM table "
        "WHERE id >= %s and id < %s" % (l_id, r_id))

    for row in mysql_cursor:
        redis_pipeline.lpush("rows", row)

    # Mark chunk as "processed" in the same pipeline where
    # we send data to redis.
    redis_pipeline.setbit("chunks", chunk_index, 1)
    redis_pipeline.execute()
```


Ok, now this looks much better! If something failed in "cursor" iterator we could 
just run this script again and it will find none processed batch. Also it seems like
we are ready to run this script using multiple processes. BUT :)

If we run this script simultaneously `chunk_index` for two processes will be set to 0 -
and these processes will get `0` from `redis_client.getbit` and will start to process this chunk **simultaneously** - which will generate data duplication. (You could think about random - but it's not a 100% solution at all).

### Step3. Time for redis connection name. 

What if we could "lock" some chunks and do not process them simultaneously. Here how `redis connection name` could help as to "lock" smth:

```python
redis_pipeline = redis_client.pipeline()
# getting all redis clients
redis_pipeline.client_list()
# set current connection name (specified as a chunk id)
redis_pipeline.client_setname("chunk_%s" % chunk_index)
client_list, _ = redis_pipeline.execute()
```

What happened there?

- We are trying to get clients information in first command
- And **then** set current connection name to some "id". (Setting connection name will be executed after client_list command)

Now we could just check if current chunk already "booked" by someone:

```python
# Continue if current chunk already "booked" by some other process
clients_names = [c['name'] for c in client_list]
if "chunk_%s" % chunk_index in clients_names:
    continue
```

Whole code:

```python
for chunk_index in range(CHUNKS_COUNT):
    # let's see if we already processed this chunk of data
    bit = redis_client.getbit("chunks", chunk_index)

    # if amount of bits set to 1 more then CHUNKS_COUNT 
    # seems like we processed everything
    if redis_client.bitcount("chunks") > CHUNKS_COUNT:
        break

    if bit == 1: continue

    redis_pipeline = redis_client.pipeline()
    # getting all redis clients
    redis_pipeline.client_list()
    # set current connection name (specified as a chunk id)
    redis_pipeline.client_setname("chunk_%s" % chunk_index)
    client_list, _ = redis_pipeline.execute()

    # Continue if current chunk already "booked" by some other process
    clients_names = [c['name'] for c in client_list]
    if "chunk_%s" % chunk_index in clients_names:
        continue

    l_id = CHUNK_SIZE * chunk_index
    r_id = CHUNK_SIZE * (chunk_index + 1)

    redis_pipeline = redis_client.pipeline()

    mysql_cursor.execute(
        "SELECT * "
        "FROM table "
        "WHERE id >= %s and id < %s" % (l_id, r_id))

    for row in mysql_cursor:
        redis_pipeline.lpush("rows", row)

    # Mark chunk as "processed" in the same pipeline where
    # we send data to redis.
    redis_pipeline.setbit("chunks", chunk_index, 1)
    redis_pipeline.execute()
```

Now you can run this script in multiple processes and it will be super "safe" and fast. 

**Quite important to know** it will work only in case if your process has one thread and one redis connection which used for naming, otherwise there is a chance that client name will be overwrite.

Enjoy :)