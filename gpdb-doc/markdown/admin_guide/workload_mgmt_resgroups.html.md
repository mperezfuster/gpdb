---
title: Using Resource Groups 
---

You use resource groups to set and enforce CPU, memory, and concurrent transaction limits in Greenplum Database. After you define a resource group, you can then assign the group to one or more Greenplum Database roles, or to an external component such as PL/Container, in order to control the resources used by those roles or components. 

When you assign a resource group to a role \(a role-based resource group\), the resource limits that you define for the group apply to all of the roles to which you assign the group. For example, the memory limit for a resource group identifies the maximum memory usage for all running transactions submitted by Greenplum Database users in all roles to which you assign the group.

Similarly, when you assign a resource group to an external component, the group limits apply to all running instances of the component. For example, if you create a resource group for a PL/Container external component, the memory limit that you define for the group specifies the maximum memory usage for all running instances of each PL/Container runtime to which you assign the group.

When using resource groups to control resources like CPU cores, review the Hyperthreading note in [Hardware and Network](../install_guide/platform-requirements-overview.html#hardware-and-network-3).

## <a id="topic8339intro"></a>Understanding Role and Component Resource Groups 

Greenplum Database supports two types of resource groups: groups that manage resources for roles, and groups that manage resources for external components such as PL/Container.

The most common application for resource groups is to manage the number of active queries that different roles may run concurrently in your Greenplum Database cluster. You can also manage the amount of CPU and memory resources that Greenplum allocates to each query.

Resource groups for roles use Linux control groups \(cgroups\) for CPU resource management. Greenplum Database implements the virtual memory assigned for these resource groups based on `query_mem` and `concurrency`.

When the user runs a query, Greenplum Database evaluates the query against a set of limits defined for the resource group. Greenplum Database runs the query immediately if the group's resource limits have not yet been reached and the query does not cause the group to exceed the concurrent transaction limit. If these conditions are not met, Greenplum Database queues the query. For example, if the maximum number of concurrent transactions for the resource group has already been reached, a subsequent query is queued and must wait until other queries complete before it runs. Greenplum Database may also run a pending query when the resource group's concurrency and memory limits are altered to large enough values.

Within a resource group for roles, transactions are evaluated on a first in, first out basis. Greenplum Database periodically assesses the active workload of the system, reallocating resources and starting/queuing jobs as necessary.

You can also use resource groups to manage the CPU and memory resources of external components such as PL/Container. Resource groups for external components use Linux cgroups to manage both the total CPU and total memory resources for the component.

> **Note** Containerized deployments of Greenplum Database might create a hierarchical set of nested cgroups to manage host system resources. The nesting of cgroups affects the Greenplum Database resource group limits for CPU percentage, CPU cores, and memory \(except for Greenplum Database external components\). The Greenplum Database resource group system resource limit is based on the quota for the parent group.

For example, Greenplum Database is running in a cgroup demo, and the Greenplum Database cgroup is nested in the cgroup demo. If the cgroup demo is configured with a CPU limit of 60% of system CPU resources and the Greenplum Database resource group CPU limit is set 90%, the Greenplum Database limit of host system CPU resources is 54% \(0.6 x 0.9\).

Nested cgroups do not affect memory limits for Greenplum Database external components such as PL/Container. Memory limits for external components can only be managed if the cgroup that is used to manage Greenplum Database resources is not nested, the cgroup is configured as a top-level cgroup.

## <a id="topic8339introattrlim"></a>Resource Group Attributes and Limits 

When you create a resource group, you provide a set of limits that determine the amount of CPU and memory resources available to the group.

Resource group attributes and limits:

|Limit Type|Description|
|----------|-----------|
|CONCURRENCY|The maximum number of concurrent transactions, including active and idle transactions, that are permitted in the resource group.|
|CPU\_HARD\_QUOTA_LIMIT|The maximum percentage of CPU resources the group can use.|
|CPU_SOFT_PRIORITY|The scheduling priority of this resource group.|
|CPUSET|The CPU cores to reserve for this resource group on the coordinator and segment hosts.|
|MEMORY\_LIMIT|The memory limit value specified for this resource group.|

> **Note** Resource limits are not enforced on `SET`, `RESET`, and `SHOW` commands.

## <a id="topic8339717179"></a>Transaction Concurrency Limit 

The `CONCURRENCY` limit controls the maximum number of concurrent transactions permitted for a resource group for roles.

> **Note** The `CONCURRENCY` limit is not applicable to resource groups for external components and must be set to zero \(0\) for such groups.

Each resource group for roles is logically divided into a fixed number of slots equal to the `CONCURRENCY` limit. Greenplum Database allocates these slots an equal, fixed percentage of memory resources.

The default `CONCURRENCY` limit value for a resource group for roles is 20.

Greenplum Database queues any transactions submitted after the resource group reaches its `CONCURRENCY` limit. When a running transaction completes, Greenplum Database un-queues and runs the earliest queued transaction if sufficient memory resources exist.

You can set the server configuration parameter [gp\_resource\_group\_bypass](../ref_guide/config_params/guc-list.html#gp_resource_group_bypass) to bypass a resource group concurrency limit, and [gp\_resource\_group\_bypass_catalog_query](../ref_guide/config_params/guc-list.html#gp_resource_group_bypass_catalog_query) to bypass a resource group concurrency limit when running queries against system catalog tables.

You can set the server configuration parameter [gp\_resource\_group\_queuing\_timeout](../ref_guide/config_params/guc-list.html) to specify the amount of time a transaction remains in the queue before Greenplum Database cancels the transaction. The default timeout is zero, Greenplum queues transactions indefinitely.

## <a id="topic833971717"></a>CPU Limits 

You configure the share of CPU resources to reserve for a resource group on the coordinator and segment hosts by assigning specific CPU core\(s\) to the group, or by identifying the percentage of segment CPU resources to allocate to the group. Greenplum Database uses the `CPUSET` and `CPU_HARD_QUOTA_LIMIT` resource group limits to identify the CPU resource allocation mode. You must specify only one of these limits when you configure a resource group.

You may employ both modes of CPU resource allocation simultaneously in your Greenplum Database cluster. You may also change the CPU resource allocation mode for a resource group at runtime.

### <a id="cpuset"></a>Assigning CPU Resources by Core 

You identify the CPU cores that you want to reserve for a resource group with the `CPUSET` property. The CPU cores that you specify must be available in the system and cannot overlap with any CPU cores that you reserved for other resource groups. \(Although Greenplum Database uses the cores that you assign to a resource group exclusively for that group, note that those CPU cores may also be used by non-Greenplum processes in the system.\)

Specify CPU cores separately for the coordinator host and segment hosts, separated by a semicolon. Use a comma-separated list of single core numbers or number intervals when you configure cores for `CPUSET`. You must enclose the core numbers/intervals in single quotes, for example, '1;1,3-4' uses core 1 on the coordinator host, and cores 1, 3, and 4 on segment hosts.

When you assign CPU cores to `CPUSET` groups, consider the following:

-   A resource group that you create with `CPUSET` uses the specified cores exclusively. If there are no running queries in the group, the reserved cores are idle and cannot be used by queries in other resource groups. Consider minimizing the number of `CPUSET` groups to avoid wasting system CPU resources.
-   Consider keeping CPU core 0 unassigned. CPU core 0 is used as a fallback mechanism in the following cases:
    -   `admin_group` and `default_group` require at least one CPU core. When all CPU cores are reserved, Greenplum Database assigns CPU core 0 to these default groups. In this situation, the resource group to which you assigned CPU core 0 shares the core with `admin_group` and `default_group`.
    -   If you restart your Greenplum Database cluster with one node replacement and the node does not have enough cores to service all `CPUSET` resource groups, the groups are automatically assigned CPU core 0 to avoid system start failure.
-   Use the lowest possible core numbers when you assign cores to resource groups. If you replace a Greenplum Database node and the new node has fewer CPU cores than the original, or if you back up the database and want to restore it on a cluster with nodes with fewer CPU cores, the operation may fail. For example, if your Greenplum Database cluster has 16 cores, assigning cores 1-7 is optimal. If you create a resource group and assign CPU core 9 to this group, database restore to an 8 core node will fail.

Resource groups that you configure with `CPUSET` have a higher priority on CPU resources. The maximum CPU resource usage percentage for all resource groups configured with `CPUSET` on a segment host is the number of CPU cores reserved divided by the number of all CPU cores, multiplied by 100.

When you configure `CPUSET` for a resource group, Greenplum Database deactivates `CPU_HARD_QUOTA_LIMIT` for the group and sets the value to -1.

> **Note** You must configure `CPUSET` for a resource group *after* you have enabled resource group-based resource management for your Greenplum Database cluster.

### <a id="cpu_hard_quota_limit"></a>Assigning CPU Resources by Percentage 

The Greenplum Database node CPU percentage is divided equally among each segment on the Greenplum node. Each resource group that you configure with a `CPU_HARD_QUOTA_LIMIT` reserves the specified percentage of the segment CPU for resource management.

Greenplum Database leverages Linux process scheduler and control groups to implement how CPU resources are assigned. How much about cgroups do we want to explain???? (Equivalences between gucs?)

Each resource group that you configure with a `CPU_HARD_QUOTA_LIMIT` reserves the specified percentage of the segment CPU for resource management..

The minimum `CPU_HARD_QUOTA_LIMIT` percentage you can specify for a resource group is 1, the maximum is 100.

The sum of `CPU_HARD_QUOTA_LIMIT`s specified for all resource groups that you define in your Greenplum Database cluster can exceed 100.

When you configure `CPU_HARD_QUOTA_LIMIT` for a resource group, Greenplum Database deactivates `CPUSET` for the group and sets the value to -1.

You may use the following parameters to limit CPU usage: 

`CPU_HARD_QUOTA_LIMIT` Specifies the maximum CPU resources that the current group can use. The sum of `CPU_HARD_QUOTA_LIMIT` specified for all resource groups in the Greenplum Databas cluster can exceed 100.

`CPU_SOFT_PRIORITY` The scheduling priority of the current group. The default value is 100, and the maximum value is 500.

Use cases:

High priority job that does not need too much resources: CPU_HARD_QUOTA_LIMIT=10, CPU_SOFT_PRIORITY=500. 
Low priority job but it needs a lot of CPU resources: CPU_HARD_QUOTA_LIMIT=100, CPU_SOFT_PRIORITY=10.  

## <a id="topic8339717"></a>Memory Limits 

When you enable resource groups, memory usage is managed at the Greenplum Database node, segment, and resource group levels. You can also manage memory at the transaction level.

The amount of memory a resource group can use is based on the value of `query_mem` of a plan. See [Greenplum Database Memory Overview](wlmgmt_intro.html) for more details.

When using resource groups, the value of `query_mem` depends on the value of the `MEMORY_LIMIT` for the specific resource group, and the value of the server configuration parameter `gp_resgroup_memory_query_fixed_mem`, if set.

- The resource limit `MEMORY_LIMIT` sets the maximum available memory reserved for this resource group. This determines the total amount of memory that all worker processes of a query can consume on a segment host during query execution.

- The server configuration parameter `gp_resgroup_memory_query_fixed_mem` sets the fixed amount of memory reserved for a query at session level. This overrides other memory limit calculations.

The amount of memory allocated per query in a resource group is calculated as:

```
query_mem = MEMORY_LIMIT / CONCURRENCY
```

- If `MEMORY_LIMIT` is set to -1, `query_mem` takes the value of `statement_mem`.
- If `gp_resgroup_memory_query_fixed_mem` is set to a value other than 0, Greenplum uses this value to set the fixed amount of memory reserved for this query (at session level).A
- If using `gp_resource_group_bypass` or `gp_resource_group_bypass` bypass query, `query_mem` takes the value of `statement_mem`.

## <a id="topic71717999"></a>Configuring and Using Resource Groups 

### <a id="topic833"></a>Prerequisites 

Greenplum Database resource groups use Linux Control Groups \(cgroups\) to manage CPU resources. Greenplum Database also uses cgroups to manage memory for resource groups for external components. With cgroups, Greenplum isolates the CPU and external component memory usage of your Greenplum processes from other processes on the node. This allows Greenplum to support CPU and external component memory usage restrictions on a per-resource-group basis.

There are two versions of cgroups: cgroup v1 and cgroup v2, which differ in the virtual file hierarchy implemented by v2. Greenplum Database uses cgroup v2 by default. For detailed information about cgroups, refer to the Control Groups documentation for your Linux distribution.

Complete the following tasks on each node in your Greenplum Database cluster to set up cgroups for use with resource groups:

1. Check your cgroup version.

1.  If not already installed, install the Control Groups operating system package on each Greenplum Database node. The command that you run to perform this task will differ based on the operating system installed on the node. You must be the superuser or have `sudo` access to run the command:
    -   Redhat/Oracle/Rocky 8.x systems:

        ```
        sudo yum install libcgroup-tools
        ```

1. If you are using Redhat 8.x, make sure that you configured the system to mount the `cgroups-v1` filesystem by default during system boot by running the following command:

    ```
    stat -fc %T /sys/fs/cgroup/
    ```

    For cgroup v1, the output is `tmpfs`.  
    If your output is `cgroup2fs`, configure the system to mount `cgroups-v1` by default during system boot by the `systemd` system and service manager:

    ```
    grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
    ```

    To add the same parameters to all kernel boot entries:

    ```
    grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
    ```

    Reboot the system for the changes to take effect.


1.  Locate the cgroups configuration file `/etc/cgconfig.conf`. You must be the superuser or have `sudo` access to edit this file:

    ```
    sudo vi /etc/cgconfig.conf
    ```

2.  Add the following configuration information to the file:

    ```
    group gpdb {
         perm {
             task {
                 uid = gpadmin;
                 gid = gpadmin;
             }
             admin {
                 uid = gpadmin;
                 gid = gpadmin;
             }
         }
         cpu {
         }
         cpuacct {
         }
         cpuset {
         }
         memory {
         }
    } 
    ```

    This content configures CPU, CPU accounting, CPU core set, and memory control groups managed by the `gpadmin` user. Greenplum Database uses the memory control group only for those resource groups created with the `cgroup` `MEMORY_AUDITOR`.

3.  Start the cgroups service on each Greenplum Database node. The command that you run to perform this task will differ based on the operating system installed on the node. You must be the superuser or have `sudo` access to run the command:
    -   Redhat/Oracle/Rocky 8.x systems:

        ```
        sudo cgconfigparser -l /etc/cgconfig.conf 
        ```

4.  Identify the `cgroup` directory mount point for the node:

    ```
    grep cgroup /proc/mounts
    ```

    The first line of output identifies the `cgroup` mount point.

5.  Verify that you set up the Greenplum Database cgroups configuration correctly by running the following commands. Replace \<cgroup\_mount\_point\> with the mount point that you identified in the previous step:

    ```
    ls -l <cgroup_mount_point>/cpu/gpdb
    ls -l <cgroup_mount_point>/cpuacct/gpdb
    ls -l <cgroup_mount_point>/cpuset/gpdb
    ls -l <cgroup_mount_point>/memory/gpdb
    ```

    If these directories exist and are owned by `gpadmin:gpadmin`, you have successfully configured cgroups for Greenplum Database CPU resource management.

6.  To automatically recreate Greenplum Database required cgroup hierarchies and parameters when your system is restarted, configure your system to enable the Linux cgroup service daemon `cgconfig.service` \(Redhat/Oracle/Rocky 8.x\) at node start-up. For example, configure one of the following cgroup service commands in your preferred service auto-start tool:
    -   Redhat/Oracle/Rocky 8.x systems:

        ```
        sudo systemctl enable cgconfig.service
        ```

        To start the service immediately \(without having to reboot\) enter:

        ```
        sudo systemctl start cgconfig.service
        ```

    You may choose a different method to recreate the Greenplum Database resource group cgroup hierarchies.


## <a id="topic8"></a>Enabling Resource Groups 

When you install Greenplum Database, no resource management policy is enabled by default. To use resource groups, set the [gp\_resource\_manager](../ref_guide/config_params/guc-list.html) server configuration parameter.

1.  Set the `gp_resource_manager` server configuration parameter to the value `"group-v1"` or `"group-v2"`, depending on the version of cgroup configured on your Linux distribution. For example:

    ```
    gpconfig -c gp_resource_manager -v "group-v2"
    ```

2.  Restart Greenplum Database:

    ```
    gpstop
    gpstart
    ```

Once enabled, any transaction submitted by a role is directed to the resource group assigned to the role, and is governed by that resource group's concurrency, memory, and CPU limits. Similarly, CPU and memory usage by an external component is governed by the CPU and memory limits configured for the resource group assigned to the component.

Greenplum Database creates two default resource groups for roles named `admin_group` and `default_group`. When you enable resources groups, any role that was not explicitly assigned a resource group is assigned the default group for the role's capability. `SUPERUSER` roles are assigned the `admin_group`, non-admin roles are assigned the group named `default_group`.

The default resource groups `admin_group` and `default_group` are created with the following resource limits:

|Limit Type|admin\_group|default\_group|
|----------|------------|--------------|
|CONCURRENCY|10|20|
|CPU_HARD_QUOTA_LIMIT| | |
|CPU_SOFT_PRIORITY| | |
|CPUSET|-1|-1|
|MEMORY\_LIMIT|10|0|

## <a id="topic10"></a>Creating Resource Groups 

When you create a resource group for a role, you provide a name and a CPU resource allocation mode (core or percentage). You can optionally provide a concurrent transaction limit, a memory limit, and a CPU soft priority. Use the [CREATE RESOURCE GROUP](../ref_guide/sql_commands/CREATE_RESOURCE_GROUP.html) command to create a new resource group.

When you create a resource group for a role, you must provide a `CPU_HARD_QUOTA_LIMIT` or `CPUSET` limit value. These limits identify the percentage of Greenplum Database CPU resources to allocate to this resource group. You may specify a `MEMORY_LIMIT` to reserve a fixed amount of memory for the resource group. 

For example, to create a resource group named *rgroup1* with a CPU limit of 20, a memory limit of 25, and CPU soft priority of 500:

```
CREATE RESOURCE GROUP rgroup1 WITH (CPU_HARD_QUOTA_LIMIT=20, MEMORY_LIMIT=25, CPU_SOFT_PRIORITY=500);
```

The CPU limit of 20 is shared by every role to which `rgroup1` is assigned. Similarly, the memory limit of 25 is shared by every role to which `rgroup1` is assigned. `rgroup1` utilizes the default `CONCURRENCY` setting of 20.

*When you create a resource group for an external component*, you must provide `CPU_HARD_QUOTA_LIMIT` or `CPUSET` and `MEMORY_LIMIT` limit values. You must also explicitly set `CONCURRENCY` to zero \(0\). For example, to create a resource group named *rgroup\_extcomp* for which you reserve CPU core 1 on coordinator and segment hosts, and assign a memory limit of 1000 MB:

```
CREATE RESOURCE GROUP rgroup_extcomp WITH (CONCURRENCY=0,
CPUSET='1;1', MEMORY_LIMIT=1000);
```

The [ALTER RESOURCE GROUP](../ref_guide/sql_commands/ALTER_RESOURCE_GROUP.html) command updates the limits of a resource group. To change the limits of a resource group, specify the new values that you want for the group. For example:

```
ALTER RESOURCE GROUP rg_role_light SET CONCURRENCY 7;
ALTER RESOURCE GROUP exec SET CPU_SOFT_LIMIT 100;
ALTER RESOURCE GROUP rgroup1 SET CPUSET '1;2,4';
```

> **Note** You cannot set or alter the `CONCURRENCY` value for the `admin_group` to zero \(0\).

The [DROP RESOURCE GROUP](../ref_guide/sql_commands/DROP_RESOURCE_GROUP.html) command drops a resource group. To drop a resource group for a role, the group cannot be assigned to any role, nor can there be any transactions active or waiting in the resource group. Dropping a resource group for an external component in which there are running instances terminates the running instances.

To drop a resource group:

```
DROP RESOURCE GROUP exec; 
```

## <a id="topic_jlz_hzg_pkb"></a>Configuring Automatic Query Termination Based on Memory Usage 

When resource groups have a global shared memory pool, the server configuration parameter [runaway\_detector\_activation\_percent](../ref_guide/config_params/guc-list.html) sets the percent of utilized global shared memory that triggers the termination of queries that are managed by resource groups that are configured to use the `vmtracker` memory auditor, such as `admin_group` and `default_group`.

Resource groups have a global shared memory pool when the sum of the `MEMORY_LIMIT` attribute values configured for all resource groups is less than 100. For example, if you have 3 resource groups configured with `MEMORY_LIMIT` values of 10 , 20, and 30, then global shared memory is 40% = 100% - \(10% + 20% + 30%\).

For information about global shared memory, see [Global Shared Memory](#topic833glob).

## <a id="topic17"></a>Assigning a Resource Group to a Role 

When you create a resource group with the default `MEMORY_AUDITOR` `vmtracker`, the group is available for assignment to one or more roles \(users\). You assign a resource group to a database role using the `RESOURCE GROUP` clause of the [CREATE ROLE](../ref_guide/sql_commands/CREATE_ROLE.html) or [ALTER ROLE](../ref_guide/sql_commands/ALTER_ROLE.html) commands. If you do not specify a resource group for a role, the role is assigned the default group for the role's capability. `SUPERUSER` roles are assigned the `admin_group`, non-admin roles are assigned the group named `default_group`.

Use the `ALTER ROLE` or `CREATE ROLE` commands to assign a resource group to a role. For example:

```
ALTER ROLE bill RESOURCE GROUP rg_light;
CREATE ROLE mary RESOURCE GROUP exec;
```

You can assign a resource group to one or more roles. If you have defined a role hierarchy, assigning a resource group to a parent role does not propagate down to the members of that role group.

> **Note** You cannot assign a resource group that you create for an external component to a role.

If you wish to remove a resource group assignment from a role and assign the role the default group, change the role's group name assignment to `NONE`. For example:

```
ALTER ROLE mary RESOURCE GROUP NONE;
```

## <a id="topic22"></a>Monitoring Resource Group Status 

Monitoring the status of your resource groups and queries may involve the following tasks:

### <a id="topic221"></a>Viewing Resource Group Limits 

The [gp\_resgroup\_config](../ref_guide/system_catalogs/catalog_ref-views.html#gp_resgroup_config) `gp_toolkit` system view displays the current limits for a resource group. To view the limits of all resource groups:

```
SELECT * FROM gp_toolkit.gp_resgroup_config;
```

### <a id="topic23"></a>Viewing Resource Group Query Status and Memory Usage 

The [gp\_resgroup\_status](../ref_guide/system_catalogs/catalog_ref-views.html#gp_resgroup_status) `gp_toolkit` system view enables you to view the status and activity of a resource group. The view displays the number of running and queued transactions. It also displays the real-time memory usage of the resource group. To view this information:

```
SELECT * FROM gp_toolkit.gp_resgroup_status;
```

#### <a id="topic23a"></a>Viewing Resource Group CPU/Memory Usage Per Host 

The [gp\_resgroup\_status\_per\_host](../ref_guide/system_catalogs/catalog_ref-views.html#gp_resgroup_status_per_host) `gp_toolkit` system view enables you to view the real-time CPU and memory usage of a resource group on a per-host basis. To view this information:

```
SELECT * FROM gp_toolkit.gp_resgroup_status_per_host;
```

### <a id="topic25"></a>Viewing the Resource Group Assigned to a Role 

To view the resource group-to-role assignments, perform the following query on the [pg\_roles](../ref_guide/system_catalogs/pg_roles.html) and [pg\_resgroup](../ref_guide/system_catalogs/pg_resgroup.html) system catalog tables:

```
SELECT rolname, rsgname FROM pg_roles, pg_resgroup
WHERE pg_roles.rolresgroup=pg_resgroup.oid;
```

### <a id="topic252525"></a>Viewing a Resource Group's Running and Pending Queries 

To view a resource group's running queries, pending queries, and how long the pending queries have been queued, examine the [pg\_stat\_activity](../ref_guide/system_catalogs/catalog_ref-views.html#pg_stat_activity) system catalog table:

```
SELECT query, waiting, rsgname, rsgqueueduration 
FROM pg_stat_activity;
```

`pg_stat_activity` displays information about the user/role that initiated a query. A query that uses an external component such as PL/Container is composed of two parts: the query operator that runs in Greenplum Database and the UDF that runs in a PL/Container instance. Greenplum Database processes the query operators under the resource group assigned to the role that initiated the query. A UDF running in a PL/Container instance runs under the resource group assigned to the PL/Container runtime. The latter is not represented in the `pg_stat_activity` view; Greenplum Database does not have any insight into how external components such as PL/Container manage memory in running instances.

### <a id="topic27"></a>Cancelling a Running or Queued Transaction in a Resource Group 

There may be cases when you want to cancel a running or queued transaction in a resource group. For example, you may want to remove a query that is waiting in the resource group queue but has not yet been run. Or, you may want to stop a running query that is taking too long to run, or one that is sitting idle in a transaction and taking up resource group transaction slots that are needed by other users.

By default, transactions can remain queued in a resource group indefinitely. If you want Greenplum Database to cancel a queued transaction after a specific amount of time, set the server configuration parameter [gp\_resource\_group\_queuing\_timeout](../ref_guide/config_params/guc-list.html#gp_resource_group_queuing_timeout). When this parameter is set to a value \(milliseconds\) greater than 0, Greenplum cancels any queued transaction when it has waited longer than the configured timeout.

To manually cancel a running or queued transaction, you must first determine the process id \(pid\) associated with the transaction. Once you have obtained the process id, you can invoke `pg_cancel_backend()` to end that process, as shown below.

For example, to view the process information associated with all statements currently active or waiting in all resource groups, run the following query. If the query returns no results, then there are no running or queued transactions in any resource group.

```
SELECT rolname, g.rsgname, pid, waiting, state, query, datname 
FROM pg_roles, gp_toolkit.gp_resgroup_status g, pg_stat_activity 
WHERE pg_roles.rolresgroup=g.groupid
AND pg_stat_activity.usename=pg_roles.rolname;
```

Sample partial query output:

```
 rolname | rsgname  | pid     | waiting | state  |          query           | datname 
---------+----------+---------+---------+--------+------------------------ -+---------
  sammy  | rg_light |  31861  |    f    | idle   | SELECT * FROM mytesttbl; | testdb
  billy  | rg_light |  31905  |    t    | active | SELECT * FROM topten;    | testdb
```

Use this output to identify the process id \(`pid`\) of the transaction you want to cancel, and then cancel the process. For example, to cancel the pending query identified in the sample output above:

```
SELECT pg_cancel_backend(31905);
```

You can provide an optional message in a second argument to `pg_cancel_backend()` to indicate to the user why the process was cancelled.

> **Note** Do not use an operating system `KILL` command to cancel any Greenplum Database process.

## <a id="moverg"></a>Moving a Query to a Different Resource Group 

A user with Greenplum Database superuser privileges can run the `gp_toolkit.pg_resgroup_move_query()` function to move a running query from one resource group to another, without stopping the query. Use this function to expedite a long-running query by moving it to a resource group with a higher resource allotment or availability.

> **Note** You can move only an active or running query to a new resource group. You cannot move a queued or pending query that is in an idle state due to concurrency or memory limits.

`pg_resgroup_move_query()` requires the process id \(pid\) of the running query, as well as the name of the resource group to which you want to move the query. The signature of the function follows:

```
pg_resgroup_move_query( pid int4, group_name text );
```

You can obtain the pid of a running query from the `pg_stat_activity` system view as described in [Cancelling a Running or Queued Transaction in a Resource Group](#topic27). Use the `gp_toolkit.gp_resgroup_status` view to list the name, id, and status of each resource group.

When you invoke `pg_resgroup_move_query()`, the query is subject to the limits configured for the destination resource group:

-   If the group has already reached its concurrent task limit, Greenplum Database queues the query until a slot opens.
-   If the destination resource group does not have enough memory available to service the query's current memory requirements, Greenplum Database returns the error: `group <group_name> doesn't have enough memory ...`. In this situation, you may choose to increase the group shared memory allotted to the destination resource group, or perhaps wait a period of time for running queries to complete and then invoke the function again.

After Greenplum moves the query, there is no way to guarantee that a query currently running in the destination resource group does not exceed the group memory quota. In this situation, one or more running queries in the destination group may fail, including the moved query. Reserve enough resource group global shared memory to minimize the potential for this scenario to occur.

`pg_resgroup_move_query()` moves only the specified query to the destination resource group. Greenplum Database assigns subsequent queries that you submit in the session to the original resource group.

## <a id="topic999"></a>Using VMware Greenplum Command Center to Manage Resource Groups

Using VMware Greenplum Command Center, an administrator can create and manage resource groups, change roles' resource groups, and create workload management rules.

Workload management assignment rules assign transactions to different resource groups based on user-defined criteria. If no assignment rule is matched, Greenplum Database assigns the transaction to the role's default resource group.

Refer to the [Greenplum Command Center documentation](http://docs.vmware.com/en/VMware-Greenplum-Command-Center/index.html) for more information about creating and managing resource groups and workload management rules.

## <a id="topic777999"></a>Resource Group Frequently Asked Questions 

### <a id="topic791"></a>CPU 

-   **Why is CPU usage lower than the `CPU_HARD_QUOTA_LIMIT` configured for the resource group?**

    You may run into this situation when a low number of queries and slices are running in the resource group, and these processes are not utilizing all of the cores on the system.

-   **Why is CPU usage for the resource group higher than the configured `CPU_HARD_QUOTA_LIMIT`?**

    This situation can occur in the following circumstances:

    -   A resource group may utilize more CPU than its `CPU_HARD_QUOTA_LIMIT` when other resource groups are idle. In this situation, Greenplum Database allocates the CPU resource of an idle resource group to a busier one. This resource group feature is called CPU burst.
    -   The operating system CPU scheduler may cause CPU usage to spike, then drop down. If you believe this might be occurring, calculate the average CPU usage within a given period of time \(for example, 5 seconds\) and use that average to determine if CPU usage is higher than the configured limit.

### <a id="topic795"></a>Memory 

-   **Why did my query return an "out of memory" error?**

    A transaction submitted in a resource group fails and exits when memory usage exceeds its fixed memory allotment, no available resource group shared memory exists, and the transaction requests more memory.

-   **Why did my query return a "memory limit reached" error?**

    Greenplum Database automatically adjusts transaction and group memory to the new settings when you use `ALTER RESOURCE GROUP` to change a resource group's memory and/or concurrency limits. An "out of memory" error may occur if you recently altered resource group attributes and there is no longer a sufficient amount of memory available for a currently running query.

-   **Why does the actual memory usage of my resource group exceed the amount configured for the group?**

    The actual memory usage of a resource group may exceed the configured amount when one or more queries running in the group is allocated memory from the global shared memory pool. \(If no global shared memory is available, queries fail and do not impact the memory resources of other resource groups.\)

    When global shared memory is available, memory usage may also exceed the configured amount when a transaction spills to disk. Greenplum Database statements continue to request memory when they start to spill to disk because:

    -   Spilling to disk requires extra memory to work.
    -   Other operators may continue to request memory.
    <br/>Memory usage grows in spill situations; when global shared memory is available, the resource group may eventually use up to 200-300% of its configured group memory limit.


### <a id="topic797"></a>Concurrency 

-   **Why is the number of running transactions lower than the `CONCURRENCY` limit configured for the resource group?**

    Greenplum Database considers memory availability before running a transaction, and will queue the transaction if there is not enough memory available to serve it. If you use `ALTER RESOURCE GROUP` to increase the `CONCURRENCY` limit for a resource group but do not also adjust memory limits, currently running transactions may be consuming all allotted memory resources for the group. When in this state, Greenplum Database queues subsequent transactions in the resource group.

-   **Why is the number of running transactions in the resource group higher than the configured `CONCURRENCY` limit?**

    The resource group may be running `SET` and `SHOW` commands, which bypass resource group transaction checks.


