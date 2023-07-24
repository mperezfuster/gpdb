# gpv greenplum

Manage the deployment of a Greenplum Database on vSphere cluster. The `gpv greenplum` command group allows you to deploy a Greenplum cluster, list the connection string to the database, and validate any deployed Greenplum cluster.

## <a id="section2"></a>Usage

```
gpv config <command>
```
## <a id="opts"></a>Commands

The available commands for `gpv config` are `deploy`, `list`, and `validate`.

### <a id="deploy"></a>deploy

Deploy Greenplum based on your configuration. The `gpv greenplum deploy` command provisions and powers on all the virtual machines, initializes the Greenplum Database cluster, and enables and starts the Greenplum Virtual Service that manages high availability.

```
gpv greenplum deploy
```

### <a id="list"></a>list

Display an endpoints to connect to Greenplum. The current available option is the PostgreSQL connection Uniform Resource Identifier (URI).

```
gpv greenplum list
```

### <a id="validate"></a>validate

Confirm that the Greenplum Database cluster deployment is successful. 

```
gpv greenplum validate
```

