# gpv greenplum


Manage the deployment of a Greenplum cluster. The gpv greenplum command group allows a user to deploy Greenplum, list the strings for connecting to Greenplum, and validate any Greenplum which has
been deployed.

## <a id="section2"></a>Usage

```
gpv config <command>
```
## <a id="opts"></a>Commands

### <a id="deploy"></a>Deploy

Deploy Greenplum based on a user's configuration. Deploy Greenplum based on a user's configuration. 

```
gpv greenplum deploy
```

The gpv greenplum deploy will perform the following operations:

provision and power-on all of the VMs based on the user's configuration
initialize Greenplum
enable and start the Greenplum Virtual Service (GPVS) for high availability

### <a id="list"></a>List

Display endpoints for clients to connect to Greenplum. The gpv greenplum list command displays the endpoints which a client must use to interact with Greenplum.  At the moment, the only endpoint which will be displayed is a postgresql connection URI.

```
gpv greenplum list
```
### <a id="validate"></a>Validate

Confirm that the Greenplum deployment is successful. The gpv greenplum validate confirms that the Greenplum cluster is functional and configured properly.

