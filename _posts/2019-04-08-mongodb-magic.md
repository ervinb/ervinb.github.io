---
layout: article
title: How to generate a MongoDB ObjectId from a date in Bash
tags: linux tips mongo
key: mongo-objectid
---

A little known fact about MongoDB object IDs is that they encode that date when the document was created. This means that
you don't have to create an index on a date field, as the `_id` field is [automatically indexed](https://docs.mongodb.com/manual/indexes/#default-id-index). Let's do it quick and dirty, sprinkled with some black magic:

```shell
# 1. get date 3 months ago
$ date=$(date --date="3 months ago" +%s)

# 2. generate the ObjectId by converting the date to hex and fill the rest with 0s
$ echo "$(printf "%x" ${date})0000000000000000"
> 5ca378720000000000000000

# 3. query the DB for files according to creation date (Atlas > Collection > globus-tracking.actions)
$ mongo
>> use mydb
>> db.collection.find({_id:{$gt: ObjectId("5cab57740000000000000000")}})
```

1. `date` has a [nifty feature](https://ss64.com/bash/date.html), where you can provide a human readable, relative string called a date string, and it will get back to you with the date at that time.

2. Mongo object IDs consist of [12 byte hexadecimal strings](). The first 4 describe the number of seconds since the epoch - the star of our show, the next 5 is a radnom value, and the last 3 bytes is a counter.
Since only the first 4 bytes are storing the date, we can fill the rest with zeros.


    We convert our date (expressed in seconds since the epoch) to Hex, by using the built-in `printf` function and the `%x` (unsigned hexadecimal) [format](https://wiki-dev.bash-hackers.org/commands/builtin/printf#format_strings).

3. We use the generated object ID to execute the [gt](https://docs.mongodb.com/manual/reference/operator/query/gt/) (greater than) query, and as a result, we will get all the documents which were created **after** the
give date.

---

Now, why would you need this?

If you don't have an index on a date field, but you need to filter out documents with a date boundary. For example:
- deleting all documents older than 30 days
- archiving all documents older than 6 months and then removing the data to save MongoDB disk space [^1]

[^1]: deleting documents will not free up the disk on the host - Mongo reserves the newly available free space for new documents. Starting from Mongo 3.2, when you [compact a collection](https://dzone.com/articles/reclaiming-disk-space-from-mongodb), then the OS will regain the free space.
