---
title: About Changes to Resource Groups in Greenplum 7
---

VMware Greenplum 7 introduces substantial changes to resource group-based resource management. 

This topic describes the Greenplum 7 resource group and compares with resource groups 6.

|Metric|Resource Groups in Greenplum 6|Resource Groups in Greenplum 7|
|------|---------------|---------------|
|Concurrency|Managed at the transaction level|Managed at the transaction level|
|CPU|Specify percentage of CPU resources or the number of CPU cores; uses Linux Control Groups|Specify percentage of CPU resources or the number of CPU cores; uses Linux Control Groups|
|Memory|Managed at the transaction level, with enhanced allocation and tracking; users cannot over-subscribe|Managed at the transaction level, with enhanced allocation and tracking; users can over-subscribe|
|Disk I/O|None|Limit the maximum read/write disk I/O throughput, and maximum read/write I/O operations per second|
|Users|Limits are applied to `SUPERUSER` and non-admin users alike<br>There are two default resource groups: `admin_group` and `default_group`|Limits are applied to `SUPERUSER`, non-admin users, and system processes of non-user classes<br>There are three default resource groups: `admin_group`, `default_group`, and `system_group`|
|Queueing|Queue when no slot is available or not enough available memory|Queue when no slot is available|
|Query Failure|Query may fail after reaching transaction fixed memory limit when no shared resource group memory exists and the transaction requests more memory|Query may fail if the allocated memory for the query surpasses the available system memory and spill limits|
|Limit Bypass|Limits are not enforced on `SET`, `RESET`, and `SHOW` commands|Limits are not enforced on `SET`, `RESET`, and `SHOW` commands. Additionally, certain queries may be configured to bypass the concurrency limit|
|External Components|Manage PL/Container CPU and memory resources|None|

The possible settings for the `gp_resource_manager` server configuration parameter have changed. They now include the following:
- `none` - Configures Greenplum Database to not use any resource manager. This is the default.
- `group` - Configures Greenplum Database to use resource groups and base resource group behavior on the cgroup v1 version of Linux cgroup functionality.
- `group-v2` - Configures Greenplum Database to use resource groups and base resource group behavior on the cgroup v2 version of Linux cgroup functionality.
- `queue` - Configures Greenplum Database to use resource queues.

Greenplum Database now includes a new server configuration parameter -- `gp_resgroup_memory_query_fixed_mem` -- which allows you to override at a session level the fixed amount of memory reserved for all queries in a resource group.
- The `gp_resgroup_status_per_segment` system view has been removed from Greenplum Database.
- The `cpu_usage` and `memory_usage` fields have been moved from the `gp_resgroup_status` system view to the `gp_resgroup_status_per_host` system view.

You may configure three new resource group attributes using the `CREATE RESOURCE GROUP` and `ALTER RESOURCE GROUP` SQL commands:
- `CPU_MAX_PERCENT`, which configures the maximum amount of CPU resources the resource group can use.
- `CPU_WEIGHT`, which configures the scheduling priority of the resource group.
- `MIN_COST`, which configures the minimum amount a query's query plan cost for the query to remain in the resource group.

The following resource group attributes have been removed:
- `CPU_RATE_LIMIT`
- `MEMORY_AUDITOR`
- `MEMORY_SPILL_RATIO`
- `MEMORY_SHARED_QUOTA`

You may now configure device I/O usage at the resource group level via the new `IO_LIMIT` resource group attribute. Specifically, you can control for processes in a resource group:
- the maximum bandwidth of read/write operations from the disk per second
- the maximum number of IO read/write operations from the disk per second

You may control this for particular tablespaces or across all tablespaces.

Refer to [Using Resource Groups](/oss/admin_guide/workload_mgmt_resgroups.html) for more information about resource groups.



