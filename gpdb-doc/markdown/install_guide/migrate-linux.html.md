---
title: Migrating VMware Greenplum from Enterprise Linux 7 to 8
---

Use this procedure to migrate a VMware Greenplum Database installation from Enterprise Linux version 7 to 8 while maintaining the existing version of Greenplum Database.

Enterprise Linux includes CentOS, Rocky, Redhat (RHEL), and Oracle Linux (OEL) as the variants supported by Greenplum. See [Platform Requirements](platform-requirements-overview.md.hbs) for a list of the supported operating systems.

Major version upgrades of Linux operating systems are always a difficult task in a Greenplum environment. You must balance the risks of different upgrade methods, downtime, and not upgrading for your business. Particularly in EL 7 to EL 8 upgrades, the task is made more difficult due to a version change in the `glibc` GNU C Library. 

There are various methods that are supported, unsupported, or currently unknown in regard to support for upgrading from EL 7 to EL 8.

Using Greenplum Copy Utility to copy from Greenplum on EL 7 to a separate Greenplum on EL 8	Supported
Using Greenplum Backup and Restore to copy from Greenplum on EL 7 to a separate Greenplum on EL 8	Supported
An inplace, simultaneous upgrade of EL 7 to EL 8 for all Greenplum hosts in a cluster	Supported (with required actions)
An inplace, batched upgrade of EL 7 to EL 8 for batches of Greenplum hosts in a cluster Unknown Support (in review)

## <a id="glib"></a>About the glibc GNU C Library Changes

Operating system vendors and in particular the authors of the GNU C library, change the locale data from time to time in minor ways to correct mistakes or add support for more languages. Between EL 7 and 8, the version of glibc changes from 2.17 to 2.28. This is a major change that impacts many languages and their collation, or sorting. A change in sorting for common languages can have a significant impact on PostgreSQL and Greenplum.

PostgreSQL, and Greenplum, use locale data provided by the operating system’s C library for sorting text. Sorting happens in a variety of contexts, including for user output, merge joins, B-tree indexes, and range partitions. In the latter two cases, sorted data is persisted to disk. If the locale data in the C library changes during the lifetime of a database, the persisted data may become inconsistent with the expected sort order, which could lead to erroneous query results and other incorrect behavior. For example, if an index is not sorted in a way that an index scan is expecting it, a query could fail to find data that is actually there, and an update could insert duplicate data that should be disallowed. Similarly, in a partitioned table, a query could look in the wrong partition and an update could write to the wrong partition. Therefore, it is essential to the correct operation of a database that the locale definitions do not change incompatibly during the lifetime of a database. This issue is well known and documented for PostgreSQL. See https://wiki.postgresql.org/wiki/Locale_data_changes.

The same issue impacts Greenplum Database and must be taken into consideration with a Major version upgrade from EL 7 to EL 8.

- All indexes involving columns of type text, varchar, char, and citext should be reindexed before the instance is put into production.
- Range-partitioned tables using those types in the partition key should be checked to verify that all rows are still in the correct partitions.
- Hash Partitions should be verified.
- To avoid downtime due to reindexing or repartitioning, consider upgrading using logical replication.
- Databases or table columns using the “C” or “POSIX” locales are not affected. All other locales are potentially affected.
- Table columns using collations with the ICU provider are not affected.

## <a id="methods"></a>Upgrade Methods

The following methods are the currently known options for how to perform the Major version upgrade from EL 7 to EL 8. 

### <a id="gpcopy"></a>Greenplum Copy Utility


The [Greenplum Copy Utility](https://docs.vmware.com/en/VMware-Greenplum-Data-Copy-Utility/index.html) is a utility for transferring data between databases in different Greenplum Database systems. 

This utility is compatible with the Greenplum Database cluster from the source and destination running on different operating systems, including EL 7 to EL 8. The glibc changes are not relevant for this migration method because the data is re-written to disk on the new cluster, which addresses any locale sorting changes. Therefore, this method is generally supported for upgrading from EL 7 to EL 8.

This upgrade method works generally by following a process of:

- Create a second Greenplum cluster on EL 8 with no data
- Fix any operating system configuration differences
- Use gpcopy to migrate data from the Source Greenplum cluster on EL 7 to the destination Greenplum cluster on EL 8
- Remove the source Greenplum cluster on EL 7

Advantages: Fully supported, optimized performance, migration issues do not impact the source cluster, it does not require table locks.
Disadvantages: Requires two, separate Greenplum clusters during the migration.

### <a id="gpbackup"></a>Greenplum Backup and Restore

Greenplum Backup and Restore supports parallel and non-parallel methods for backing up and restoring databases. 

https://docs.vmware.com/en/VMware-Greenplum-Backup-and-Restore/1.28/greenplum-backup-and-restore/backup-overview.html

The utility is compatible with the Greenplum Database cluster from the source and destination running on different operating systems, included EL 7 to EL 8. The glibc changes are not relevant for this migration method because the data is re-written to disk on the new cluster, which addresses any locale sorting changes. Therefore, this method is generally supported for upgrading from EL 7 to EL 8.

Greenplum Backup and Restore supports many different options for storage locations. Including local, public cloud storage such as S3, and Dell EMC DataDomain through the use of gpbackup storage plugins (https://docs.vmware.com/en/VMware-Greenplum-Backup-and-Restore/1.28/greenplum-backup-and-restore/admin_guide-managing-backup-plugins.html).  Any of the supported options for storage locations to perform the data transfer are supported for the EL 7 to EL 8 upgrade.

This upgrade method works generally by following a process of:

Create a second Greenplum cluster on EL 8 with no data
Fix any operating system configuration differences
Use gpbackup to take a full backup of the Source Greenplum cluster on EL 7
Restore the backup with gprestore to the destination Greenplum cluster on EL 8
Remove the source Greenplum cluster on EL 7

Advantages: fully supported, options for storage locations, migration issues do not impact the source cluster.
Disadvantages: requires two separate Greenplum clusters during the migration, generally slower than Greenplum Copy, it requires table locks to perform a full backup.

### <a id="simultaneous"></a>Simultaneous, In-Place Upgrade

Redhat and Oracle Linux both support options for in-place upgrade of the operating system using the Leapp utility. While supported, it is typically avoided by most IT departments due to the risk and complexity of the process.

https://access.redhat.com/articles/4263361

https://docs.oracle.com/en/operating-systems/oracle-linux/8/leapp/#Oracle-Linux-8

This upgrade method works generally by following a process of:

Run precheck script to get details (name and size) of objects which can potentially have data impact from upgrade
Stop Greenplum
Use Leapp to in-place upgrade the operating system
Fix any operating system configuration differences
Start Greenplum and follow required steps for fixing data impacted by the glibc locale sorting changes
Postupgrade script will be provided to rewrite the potentially affected objects

Advantages: it does not require two different Greenplum clusters.
Disadvantages: risk of performing in-place operating system upgrade, no downgrade options after any issues, issues could leave your cluster in a non-operating state, requires extra steps after upgrade to address glibc changes, Greenplum downtime for the entire process.

#### <a id="reindex"></a>Reindex for Collation Changes

When you perform an upgrade using the simultaneous, in-place method, you must perform additional steps in order to fix the data impacted by the glibc locale sorting changes.

All indexes involving columns of type text, varchar, char, and citext should be reindexed before the instance is put into production.
Indexes may be reindexed with the REINDEX command. https://docs.vmware.com/en/VMware-Greenplum/6/greenplum-database/ref_guide-sql_commands-REINDEX.html
Use this SQL query in each database to find out which indexes are affected:

```
// 7X:
SELECT indrelid::regclass::text, indexrelid::regclass::text, collname, pg_get_indexdef(indexrelid) 
FROM (SELECT indexrelid, indrelid, indcollation[i] coll FROM pg_index, generate_subscripts(indcollation, 1) g(i)) s 
  JOIN pg_collation c ON coll=c.oid
WHERE collprovider IN ('d', 'c') AND collname NOT IN ('C', 'POSIX');
 
// 6X:
SELECT coll, indexrelid, indrelid::regclass::text, indexrelid::regclass::text, collname, pg_get_indexdef(indexrelid)
FROM (SELECT indexrelid, indrelid, indcollation[i] coll FROM pg_index, generate_subscripts(indcollation, 1) g(i)) s
JOIN pg_collation c ON coll=c.oid
WHERE collname NOT IN ('C', 'POSIX');
```


Range-partitioned tables using those types in the partition key should be checked to verify that all rows are still in the correct partitions.
[ Greenplum provided tool / steps to identify and perform this step ]
Use this SQL query in each database to find out which range-partitioned tables are affected:

```

// 7X:
SELECT partrelid::regclass::text
FROM (SELECT partrelid, partcollation[i] coll FROM pg_partitioned_table, generate_subscripts(partcollation, 1) g(i)) s
JOIN pg_collation c ON coll=c.oid
WHERE collprovider IN ('d', 'c') AND collname NOT IN ('C', 'POSIX');
 
// 6X:
SELECT
  coll,
  attrelid::regclass::text,
  attname,
  attnum
FROM
  (
    select
      t.attcollation coll,
      t.attrelid,
      t.attname,
      t.attnum
    from
      pg_partition p
      join pg_attribute t on p.parrelid = t.attrelid
      and t.attnum = ANY(p.paratts :: smallint[])
  ) s
  JOIN pg_collation c ON coll = c.oid
WHERE
  collname NOT IN ('C', 'POSIX');
```

Hash distribution should be verified.
[ Greenplum provided tool / steps to identify and perform this step ]

### <a id="os_config"></a>Operating System Configuration Differences

When you prepare your operating system environment for Greenplum Database software installation, there are different configuration options depending on the version of your operating system. Documentation already exists in the pages [Configuring Your Systems](https://docs.vmware.com/en/VMware-Greenplum/6/greenplum-database/install_guide-prep_os.html) and [Using Resource Groups](https://docs.vmware.com/en/VMware-Greenplum/6/greenplum-database/admin_guide-workload_mgmt_resgroups.html#topic71717999). This section summarizes the main changes to take into consideration when you upgrade from EL 7 to EL 8, regardless of the upgrade method you use.

#### <a id="xfs"></a>XFS Mount Options

XFS is the preferred data storage file system on Linux platforms. Use the mount command with the following recommended XFS mount options. The nobarrier option is not supported on EL 8 or Ubuntu systems. Use only the options:

```
rw,nodev,noatime,inode64
```

#### <a id="diskio"></a>Disk I/O Settings

The Linux disk scheduler orders the I/O requests submitted to a storage device, controlling the way the kernel commits reads and writes to disk. A typical Linux disk I/O scheduler supports multiple access policies. The optimal policy selection depends on the underlying storage infrastructure. 

|Storage Device Type|Recommended Scheduler Policy|
|------|---------------|
|Non-Volatile Memory Express (NVMe)|none|
|Solid-State Drives (SSD)|none|
|Other|mq-deadline|

To specify the I/O scheduler at boot time for EL 8 you must either use TuneD or uDev rules. See Redhat documentation for full details: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/setting-the-disk-scheduler_monitoring-and-managing-system-status-and-performance. 

#### <a id="ntp"></a>Synchronizing System Clocks

You should use NTP (Network Time Protocol) to synchronize the system clocks on all hosts that comprise your Greenplum Database system. See www.ntp.org for more information about NTP.

NTP on the segment hosts should either be configured to use the master host as the primary time source, and the standby master as the secondary time source or all hosts should be configured to use the same external, primary time source.

On EL 8, NTP should be configured with the Chorny service. See redhat documentation for full details https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/using-chrony_configuring-basic-system-settings

#### <a id="resgroups"></a>Configuring and Using Resource Groups

Greenplum Database resource groups use Linux Control Groups (cgroups) to manage CPU resources. Greenplum Database also uses cgroups to manage memory for resource groups for external components. With cgroups, Greenplum isolates the CPU and external component memory usage of your Greenplum processes from other processes on the node. This allows Greenplum to support CPU and external component memory usage restrictions on a per-resource-group basis.

If you are using Redhat 8.x, make sure that you configured the system to mount the cgroups-v1 filesystem by default during system boot by running the following command:

stat -fc %T /sys/fs/cgroup/
For cgroup v1, the output is tmpfs.
If your output is cgroup2fs, configure the system to mount cgroups-v1 by default during system boot by the systemd system and service manager:

grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
To add the same parameters to all kernel boot entries:

grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
Reboot the system for the changes to take effect.
