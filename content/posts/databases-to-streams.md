---
title: "Databases to Streams"
# featured_image: "images/redis_polling_agent.png"
date: 2019-03-24T09:23:40+11:00
draft: false
---

Redis streams, added in version 5, provide a very powerful yet simple tool.
One application of them is to propagate changes instantly from producer to consumer(s).

In my organisation, many of our Sources of Truth are legacy SQL databases, which we have
little control over. We need to consume changes in a range of downstream systems, and the
pre-existing solution was to establish point-to-point services which would poll the databases,
and write the changed records into another database. I know, bad in a few ways. Redis
to the rescue!

# How it works

Say we want to generate a Stream of customer data, where a new message is XADDed to redis
whenever something changes in our upstream database. Assume it is too dumb to support a
LISTEN/NOTIFY pattern, too.

(I will gloss over the initial poll - I can go into detail in another post, if there is interest)

{{< figure src="/plant/redis_polling_agent.png" >}}

1. We create a polling agent, configured to periodically execute a SQL query against the source,
   using some means to identify customers with updates since time X. (If your query spans multiple
   tables, this may involve calculating the MAX of various timestamps).

1. Each polling cycle, the agent queries records in the database which have been recently modified.
   'Recent' is defined as the oldest (lowest-scored) entry in our Sorted Set.
   To cater for potentially
   long-running transactions, we extend our window back a period of time, sufficient to catch these.
   This is ugly, but unavoidable.

1. Now for the fun part: using a Lua script, we use a Sorted Set (`KEYS[1]`) to
   deduplicate the messages,
   before inserting them into a Stream (`KEYS[3]`).
   Notice that this command will also record the new
   message identifier (`KEYS[2]`) along with its timestamp (`epoch`) in the sorted set.
   {{< highlight lua >}}
   local maxlen = table.remove(ARGV)
   local epoch = table.remove(ARGV)
   local variant = table.remove(ARGV)
   local score = redis.call("ZSCORE", KEYS[1], KEYS[2])
   if not score or tonumber(score) < tonumber(epoch) then
   redis.call("ZADD", KEYS[1], epoch, KEYS[2])
   return redis.call("XADD", KEYS[3], {} '\*', "payload", ARGV[1], "variant", variant, "epoch", epoch, "pkey", KEYS[2])
   end
   return
   {{< / highlight >}}

1. After the new batch of records are processed by the Lua script, `ZREMREANGEBYSCORE` is used to
   discard members of the sorted set older than the oldest message we just processed.
1. Sleep for a configured period, then re-poll the source database.
