# Replication in Appointal

## What is MongoDB replication?
Replication means keeping **copies of the same data on several MongoDB servers**. If one server fails, another one still has the data, so the application keeps working.

## Key terms
| Term | Meaning |
|---|---|
| **Replica set** | A group of MongoDB servers that hold the same data (usually 3). |
| **Primary** | The one server that accepts all writes (insert, update, delete). Reads normally go here too. |
| **Secondary** | Servers that continuously copy the primary's changes. They hold identical data and can be used for reads if configured. |
| **Automatic failover** | If the primary stops working, the remaining members hold an *election* and one secondary becomes the new primary, usually within seconds, without manual work. |

## How it works
```
Appointal app --writes--> PRIMARY --copies changes--> SECONDARY 1
                              |
                              +--copies changes--> SECONDARY 2

If PRIMARY fails:  SECONDARY 1 or 2 is elected as the new PRIMARY.
```
The primary records every change in a log (the *oplog*). Secondaries read this log and apply the same changes.

## Why replication improves availability
- **High availability:** the system keeps running if one server crashes.
- **Data safety:** several copies exist, so a single hardware failure doesn't lose data.
- **Read scaling (optional):** some reads can be served by secondaries.
- **Less maintenance downtime:** servers can be updated one at a time.

Replication is **not a backup**. If someone deletes data by mistake, the deletion is copied to the secondaries too. That is why we also use backups (see `BACKUP_AND_RECOVERY.md`).

## Replication in Appointal (MongoDB Atlas)
Appointal uses MongoDB Atlas. Atlas clusters are **replica sets by default** (normally three nodes), and Atlas manages failover for you. You don't write any replication code in Java.

The `mongodb+srv://` connection string lists the cluster's members automatically. The MongoDB driver used by Spring Boot finds the current primary and reconnects after a failover.

To see it in Atlas: open your cluster -> *Overview*; it shows the primary and secondary nodes.

## Optional: local replica set (for learning only)
Not required for this project. MongoDB's documentation explains how to start three local `mongod` processes with `--replSet` and run `rs.initiate()`. Atlas does all of this for you.
