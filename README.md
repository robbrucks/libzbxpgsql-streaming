# libzbxpgsql-streaming
Streaming Monitoring Add-On for Libzbxpgsql v1.1

## Setup
* *This Document assumes you have the libzbxpgsql v1.1 module installed, configured, and already succesfully monitoring your database clusters (instances)*

1. Import the template file "postgresql\_streaming\_template.xml" into Zabbix via the UI
1. Copy the following files into the `/etc/zabbix` directory on your master and slave DB servers
  * `libzbxpgsql.conf`
  * `streaming.conf`
1. Execute the SQL script `replication_pump_func.sql` on each *master* DB cluster against the same database as defined in your {$PG\_DB} macro in Zabbix (the `postgres` database by default)

  ```
  psql -f replication_pump_func.sql -U postgres -d postgres
  ```

1. Link the `Template App PostgreSQL Streaming` template to each of your master and slave DB hosts via the Zabbix UI
1. Restart the zabbix-agent on your DB servers

## What is monitored
* Count of WAL log bytes waiting to be applied on each connected slave ("lag bytes"; measured on the master)
* Number of seconds a slave is behind the master ("lag seconds"; measured on each slave)
* Whether or not replication has been paused on a slave
* If "lag bytes" exceeds the value of Zabbix macro variable "{$PG\_ALRT\_SLAVE\_LAG\_BYTES}" (default 100mb) then an alert occurs in Zabbix
* If "lag seconds" exceeds the value of Zabbix macro variable "{$PG\_ALRT\_SLAVE\_LAG\_SECS}" (default 300 seconds) then an alert occurs in Zabbix
* If replication is paused on a slave, then an alert occurs in Zabbix

## What's up with the "Replication Master Log Pump" thing?
Get ready for a lengthy explanation...

On a master/slave setup using streaming replication, selecting from the `pg_last_xact_replay_timestamp()` function on the slave will report the timestamp of the last update replayed on the slave.  This works fine when the master has constant update activity, but if there is a period of time where there are no updates to be streamed to the slave then the timestamp returned by this function will not get updated. This can cause the slave to appear to be significantly behind the master, based on this timestamp, despite actually being completely up to date with the master.

I discovered that issuing a simple `NOTIFY` command on the master will cause the notification to be streamed to the slaves, even if there are no corresponding listeners. The documentation indicates that the notification is discarded if there are no corresponding listeners, so this appears to be a relatively harmless and lightweight method to force the replay timestamp to be updated on the slaves.

Unfortunately libzbxpgsql cannot issue a `NOTIFY` command, so I had to implement it using a function, which libzbxpgsql CAN execute.

So the `PostgreSQL Streaming Replication Master Log Pump` item executes this function every 30 seconds. It "pumps" the WAL log stream.

By issuing the `NOTIFY` every 30 seconds I can ensure that the replay timestamp is updated at least that frequently on all the slaves. So even if a master goes "quiet" it will still be forced to stream every 30 seconds and update the timestamp on the slaves. Then if the timestamp fails to get updated I can alert that there is a problem with replication. As long as the slave database is in recovery mode then this metric will be accurate.

## But I can alert on lag bytes...
Yes, this template also measures the lag bytes as reported by the master in the `pg_stat_replication` view, and it will alert if lag bytes becomes high. But that view has a critical weakness: if communication with the slave is lost, the corresponding row in `pg_stat_replication` is immediately deleted for that slave, and you will not know how far behind replication is. Since Zabbix won't get an updated metric from the missing row, Zabbix can't alert that the slave is falling behind.

## What am I trying to solve here?
I'm trying to solve one of the more commonly encountered problems with replication: How can I tell that a communication issue has stalled streaming replication?

I think I've solved it - but please let me know if I've got something wrong...

