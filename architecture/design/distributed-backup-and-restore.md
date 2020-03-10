# Distributed Backup And Restore

The traditional way to take backups in an RDBMS is to *dump* the data across all (or the desired set of) the tables in the database by performing a scan operation. However, in YugabyteDB - a *distributed* SQL database that is often considered for its massive scalability, the data set could become quite large, making a scan-based backup practically infeasible. The distributed backups and restore feature is aimed at making backups and restores very efficient even on large data sets.

> **Note:** An **in-cluster snapshot** refers to a read-only copy of the database created by creating a list of immutable files. A **backup** generally is an off-cluster copy of a snapshot of the database. For the purposes of this design, we will be using backups to refer to snapshots with the understanding that copying a snapshot off the cluster is a user driven exercise.

# Goals

#### 1. ACID consistency
* A snapshot should be consistent with on-going transactions across different rows and tables
* Example: a transaction should either be completely included into the snapshot or fully excluded
#### 2. Point-in-time consistency
* Example: a newer transaction should not be included without all prior transactions being included in the snapshot
#### 3. Recency
* Ensure that all the changes made before the snapshot request are included in the snapshot
#### 4. Efficiency for large data set sizes
* Time to perform an in-cluster snapshot should not depend on the dataset size
* This implies data should not be scanned in order to perform a backup
#### 5. No dependency on cluster configuration or deployment topology
* Example: possible to restore snapshots from a cluster into any target cluster (with different number/type of nodes)
#### 6. Support for secure clusters
* Authentication and authorization - support backing up users, roles and permissions
* Encryption of data at rest - support backuping up data with encryption preserved as well as un-encrypted
#### 7. Ease of use
* This functionality should work across all the YugabyteDB APIs (YSQL and YCQL)
* Extend this functionality to the YSQL grammar to simplify this process


# Design

### 1. Initiating a snapshot
The snapshot will be initiated from the YSQL or the YCQL query layer, because this would have a dependency on the schema. In the case of either API, snapshots can be performed at the following granularities:
* An entire database
* A set of tables and all their relations (indexes, unique constraints, `SERIAL` types for auto-generated ids, etc)
* A single table

Note that the current plan is to support the following levels of atomicity in a backup:
* **YSQL snapshots:** An entire database would get backed up. This would include all relations such as indexes, unique constraints, `SERIAL` types for auto-generated ids, foreign keys, etc.
* **YCQL snapshots:** A table and it's direct relations would be backed up.

### 2. API call to the YB-Master
The request to perform a point-in-time snapshot of the cluster is initiated via the following API call to the YB-Master leader.
```
CreateSnapshot(<list of tablets>)
```

This API request specifies the set of tablets that need to be backed up. The set of tablets determines the scope of what is being backed up - for example, snapshots can be performed at the following granularities:
* An entire database
* A set of tables and all their relations (indexes, unique constraints, `SERIAL` types for auto-generated ids, etc)
* A single table

### 3. Picking the snapshot timestamp
In order to ensure that a snapshot is consistent, it has to performed as of a fixed, cluster-wide timestamp, referred to subsequently as the snapshot hybrid timestamp, or the `snapshot-timestamp` for short. Upon receiving a `PerformSnapshot()` API call, the YB-Master leader first picks a suitable `snapshot-timestamp`.

It is important to ensure that there are no changes to the data in the database older than the `snapshot-timestamp` after a snapshot is taken, while ensuring that all the updates made before the snapshot request are included. Based on how the distributed transactions algorithm of YugabyteDB works, we already know that in order for all recently written transaction to be included, we would need the timestamp that is computed as follows:
```
snapshot-timestamp = current_physical_time_on_node() + max_clock_skew_in_cluster
```

Further, in order for the database to ensure that no updates older than the `snapshot-timestamp` will be applied, we would need to wait till the hybrid logical time on the YB-Master leader advances to atleast the following:
```
snapshot-start-time = snapshot-timestamp + max_clock_skew_in_cluster
```

> **Note:** It is possible to choose the `snapshot-timestamp` as `current_physical_time_on_node() - max_clock_skew_in_cluster` and start the `snapshot-start-time` as `current_physical_time_on_node()`. However, this choice violate the *recency* design goal, because some of the updates performed in the `( current_physical_time_on_node() - max_clock_skew_in_cluster, current_physical_time_on_node() + max_clock_skew_in_cluster )` time window might not make it into the snapshot. 

### 4. Backing up system catalog

The system catalog would get backed up in the YB-Master, which includes the schema of the database at the time of the backup. This backup would be very similar to what is captured by the output of:
```
ysql_dump --schema-only
```

### 5. Snapshot all tablets at snapshot-time

The YB-Master leader sends RPCs to all YB-TServers asking them to create a consistent snapshot at the hybrid logical time represented by `snapshot-timestamp`. These API calls would look as follows:
```
YB-Master leader --> each of the YB-TServers (per tablet):
    SnapshotTabletAtHybridTimestamp(snapshot-timestamp, tablet)
```

Upon receiving this RPC call, the YB-TServers would resolve all provisional writes (writes that are a part of recent transactions) by consulting the transaction status table. Note that by virtue of waiting out the max clock skew, none of the currently `PENDING` transaction can get a commit timestamp lower than `snapshot-timestamp`.

Once all the `APPLY` records for the provisional writes are replicated across the tablet peers, the tablet leader replicates a `SNAPSHOT` record containing the `snapshot-timestamp`. Applying this record into the tablet Raft log creates a DocDB checkpoint which includes the following:
* A hardlink based snapshot of all the files (which are immutable).
* Special snapshot metadata for each of the above files indicating all timestamps higher than the `snapshot-timestamp` should be ignored when reading these files.

The above steps ensure that the data does not have to be rewritten, making the operation efficient for larger data sets.


[![Analytics](https://yugabyte.appspot.com/UA-104956980-4/architecture/design/distributed-backup-and-restore.md?pixel&useReferer)](https://github.com/yugabyte/ga-beacon)