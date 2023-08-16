# gpv 

Configures, deploys, and manages Greenplum Database clusters on top of a virtual platform such as VMware vSphere.

## <a id="section2"></a>Synopsis 

```
gpv base <base_options>
gpv config <config_options>
gpv greenplum <greenplum_options>
gpv version
gpv -h | --help
```

## <a id="section3"></a>Description 

The `gpv` utility automates the deployment of a Greenplum Database cluster on top of virtual platforms such as vSphere. You use `gpv` to specify a Greenplum cluster's configuration; automate the deployment of a base virtual machine, clone the base virtual machine to create a cluster, and initialize a Greenplum Database on the set of virtual machines. See [VMware Greenplum on vSphere](/gpvirtual/vsphere/index.html) for more details.

## <a id="section4"></a>Options

[gpv base](gpv-base.html)
:   The base virtual machine commands.

[gpv config](gpv-config.html)
:   The commands to manage the configuration of the Greenplum deployment.

[gpv greenplum](gpv-greenplum.html)
:   The Greenplum initialization and validation commands.

-version
:   Print the version information for the `gpv` utility.

-h, --help
:   Display the online help. 

## <a id="info"></a>Required Inputs

The following table lists all the information you require in order to configure the a base virtual machine template using the `gpv` utility. Be sure you have this information available before you start with the configuration.

|Configuration|Description|
|-|-| 
|**vSphere**|**Configuration parameters for vSphere**|
|vSphere admin username|An administrator account with enough permissions to deploy the Greenplum cluster|
|vSphere admin password|The password for the administrator account| 
|vCenter address|The FQDN or IP address of the vCenter (do not include `https://`)|
|Datacenter name|The virtual data center name|
|Compute cluster name|The virtual compute cluster name|
|Storage type |The storage provider type (`powerflex` or `vsan`)|
|Storage name|The datastore name for this deployment|
|vSphere distributed switch MTU size|The Maximum Transmission Unit (MTU) configured for the vSphere distributed switch*|
|**Database**|**Configuration parameters for Greenplum Database**|
|Deployment type|Greenplum deployment type (`mirrored` or `mirrorless`)|
|Greenplum database cluster prefix |Prefix to prepend to the resource pool and virtual machine names. Default is `gpv`|
|Number of Primay Segment virtual machines | The number of virtual machines for the primary segments. <br/> - For a `mirrored` deployment type the value is `number of primary segments * 2 + 2`. <br/> - For a `mirrorless` deployment type the value is `number of primary segments + 1`|
|**Base-vm**|**Configuration parameters for the base virtual machine**|
|Base VM network type |Available options are `dhcp` or `static` IP for the **base virtual machine only**. <br/> - For `dhcp`, the DHCP server assigns the network settings <br/> - For `static` IP, you must specify the virtual machine IP, the gateway IP, and the netmask|
|Base VM IP| The IP to use if the network type is `static`|
|Base VM gateway IP|The gateway to use if the network type is `static`|
|Base VM netmask|The Netmask to use if the network type is `static`|
|**gp-virtual-external**|**Configuration parameters for the routable network**|
|gp-virtual-external CIDR|The Classless Inter-Domain Routing (CIDR) of the routable network connected to the `mdw` and `smdw` virtual machines to provide the Greenplum database service to the user clients|
|gp-virtual-external available IPs|The space separated IP addresses used for `mdw` and `smdw`. The second IP (for `smdw`) is only required for `mirrored` deployment.|
|gp-virtual-external DNS servers|Space separated IP addresses of the DNS servers on the routable network|
|gp-virtual-external NTP servers|Space separated addresses of the NTP servers|
|gp-virtual-external gateway IP|The gateway IP address|
|**gp-virtual-internal**|**Configuration parameters for the Greenplum internal network**|
|gp-virtual-internal CIDR|The internal network The Classless Inter-Domain Routing (CIDR) for the interconnect communications between Greenplum virtual machines|
|**gp-virtual-etl-bar**|**Configuration parameters for the ETL-BAR network**|
|gp-virtual-etl-bar CIDR|The network The Classless Inter-Domain Routing (CIDR) for Extraction Transformation Load (`etl`) and/or Backup and Restore (`bar`) network. Usually this network is not routable to the database user clients, but routable to the backup or staging servers. This network allows access to all segments within the Greenplum cluster to ensure maximum throughput for heavy data movement workloads.|
|**Virtual machine options**|**Configuration parameters for the virtual machines**|
|base-VM-name|The name of the base virtual machine, provisioned in the next section|
|boot password for the root user|The password for the `root` user on the base virtual machine|
|boot password for the gpadmin user|The password for the `gpadmin` user on the base virtual machine. The `gpadmin` user is required for the Greenplum cluster initialization and management.|
|number of primary segment VMs|The number of virtual machines for the primary segments. <br/> - for `mirrored` the total number of virtual machines is `number of primary segments * 2 + 2`. <br/> - for `mirrorless` the total number of virtual machines is `number of primary segments + 1`.|
|||

*Greenplum performs best with jumbo frames. The recommended MTU is 9000 if the vSphere distributed switch supports it. If it is less than 9000, you must adjust the server configuration parameter [gp_max_packet_size](../../ref_guide/config_params/guc-list.html#gp_max_packet_size) manually. See [Configuring Your Systems](../../install_guide/prep_os.html#networking) for more information about MTU.

## <a id="exs"></a>Examples 

**Display `gpv` version**

``` 
gpv version
```

**Display the online help**

``` 
gpv --help
```

