---
layout: post
title: Restoring CloudSQL from a point-in-time backup
subtitle: Fun with MySQL binlogs
tags: [binlog, clousql, mysql]
image: /assets/img/mysql.png
comments: true
---

We've had a problem. We need to restore our CloudSQL MySQL database using point-in-time recovery. There's a runbook of how to do that, right?
This is a technical write up of the next few hours of learning how to actually do this in a sane way.

# How did we get here?

At GoCardless, we have a service that runs on a CloudSQL powered MySQL instance. Something had *gone wrong* and we needed to recover some deleted data, ideally 'just before' it was deleted. GCP provides two backup types: a scheduled one and a 'point in time' one. We tried restoring from one of the scheduled daily backups as a starting point, but that failed instantly. Instead, we started looking at 'point in time recovery'. To do this, we needed to provide a location in the binary log file.

# A binary log file, you say?

There's a detailed overview in the [MySQL Docs](https://dev.mysql.com/doc/internals/en/binary-log-overview.html#:~:text=The%20binary%20log%20is%20a,was%20introduced%20in%20MySQL%203.23.), but the TLDR is:

The binary log is a set of log files that contains a list of statements which could have updated data in the MySQL instance (it ignores reads). It includes a timestamp, a duration (how long the query took to complete) and some internal information that MySQL uses to do its thing. It's used by the master replica to send events to any secondary servers to keep them up to date (via something called a relay log), but it can also be used for data recovery (i.e. to replay everything that happened to a database up until a particular point).


# This is all sounding sensible

So the fun part here is 'how do I get from a timestamp to a binary log file location'. We knew that we wanted to restore to just before 1:02pm - we called it 1pm to be on the safe side. Some StackOverflow-ing later, we got to the following set of steps.

### 1. mysql-server

To do anything useful with the binary logs, you need to be able to parse them. MySQL provides a nice utility for this: [`mysqlbinlog`](https://dev.mysql.com/doc/refman/8.0/en/mysqlbinlog.html). Understandably, it's packaged inside the CLI utility `mysql-server`. Our application pods (like most such pods, I would expect) have `mysql-client` installed by not `mysql-server`. So step 1 was adding a line to our application pod Dockerfile
```
apt-get install mysql-server
```

Note that it may have been possible to SSH onto the CloudSQL pod itself, but we didn't have the permissions to do that so we chose to use the application pods instead.

### 2. find the correct binlog file

MySQL partitions its binary log into files: exactly how you would expect. Each file contains all the updates in a particular time frame. The `mysqlbinlog` utility doesn't allow you to query across files, so our first task was to find the correct binlog file.

First, get a list of binlog files (you can run this is a normal mysql client)
```
mysql > SHOW BINARY LOGS

+---------------+-----------+
| Log_name      | File_size |
+---------------+-----------+
| binlog.000015 |    724935 |
| binlog.000016 |    733481 |
+---------------+-----------+
```

Then, haphazardly check the first events for each file until you find the file that has your desired timestamp inside. In our case we only had to check a couple of files so this was fine - if you have large volumes you may want to do some piping to make this more automated.
```
mysqlbinlog \
  -h "$DB_DEFAULT_HOST"  \
  -u "$DB_DEFAULT_USER" \
  --read-from-remote-server \
  --stop-position=500 \
  binlog.000015 \
  | grep "end_log_pos" -m 1

=> #201230 12:23:59 server id 12345  end_log_pos 123 CRC32 0xf16c2470 	Start: binlog v 4, server v 5.7.23-log created 201230 12:23:59 at startup
```
You can see the timestamp at the start of this line - this indicates when this binlog file starts.
We need the `read-from-remote-server` flag because we are running this utility in our application code. If you are on the server pod itself, you won't need this.
You need to specify a host and user (and possibly also a password) - use the docs to guide you.
The `--stop-position` is just so we don't read the entire file. 500 is an arbitrary length which we found worked to make sure you get the first proper event.
The `grep` just filters out lots of the noise in the log file - you might want to look without it to see what the whole file looks like.

### 3. find the correct binlog location

Now that we know which file we're in, we just need to get the `location` or `position` of the line we want to restore up to. We can do that by defining a time range and asking for all binlog entries from our file in that time range:

```
mysqlbinlog \
  -h "$DB_DEFAULT_HOST" \
  -u "$DB_DEFAULT_USER" \
  --read-from-remote-server \
  --start-datetime="2020-12-30 12:24:02" \
  --stop-datetime="2020-12-30 12:24:05" \
  binlog.000015 \
  | grep "end_log_pos"

=>

#201230 12:23:59 server id 12345  end_log_pos 123 CRC32 0xf16c2470 	Start: binlog v 4, server v 5.7.23-log created 201230 12:23:59 at startup
```

What you want is the `end_log_pos` - that is the 'location' of this particular binary log entry.

### 4. let CloudSQL do its thing

Follow the instructions in the [Google docs](https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr#perform-pitr) to specify your binlog file and location to trigger a point-in-time recovery. Then go for a coffee (ours took about 4 hours) and you should have a database as it was at your original timestamp! Congrats 🎉

## Post script

Note that, as you can see in Google's own docs on [how to do this](https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr#coordinates), you can see binary log events from a normal mysql client by running `SHOW BINLOG EVENTS`. This gets you lots of useful information, but does not (bizzarely) include the timestamp making it totally useless for our use case. The queries that it stores are also hard to search through quickly unless you have the xid of the transaction you are looking for.
