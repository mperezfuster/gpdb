# gpv config

Manage the configuration of a Greenplum Database on vSphere cluster. The `gpv config` command group allows you to configure a Greenplum cluster, import an external configuration, and list the current configuration.

## <a id="section2"></a>Usage

```
gpv config <command>
```

## <a id="opts"></a>Commands

The available commands for `gpv config` are `init`, `list`, and `set`.

### <a id="init"></a>init

Initialize a configuration interactively or import an external configuration from a file. The `gpv config init` command creates the configuration for deploying a Greenplum Database cluster. You may speficy the configuration parameters entering each value interactively, or from an existing configuration yaml file, `file_name`.

```
gpv config init [<file_name>]
```

#### <a id="ex_init"></a>Examples

Start a new configuration from scratch: 

```
gpv config init
```

Import an existing configuration from a file: 

```
gpv config init /tmp/config.yaml
```

### <a id="list"></a>list

List the current configuration settings. The `gpv config list` command displays the current configuration for deploying Greenplum on vSphere.

### <a id="set"></a>set

Set an individual configuration setting. 

```
gpv config set <area>
```

Where `area` can be one of the following:

#### <a id="database"></a>database

Configure the settings for the Greenplum Database.

```
gpv config set database <setting>
```

Settings:

deployment-type <type_name>
:   Specifies whether the Greenplum deployment uses mirror segments or not. Valid values of `<type_name>` include `mirrored` and `mirrorless`. For example: `gpv config set database deployment-type mirrored`. 

prefix <prefix_name>
:   Specifies a label which serves as a prefix for the names of the resource pool and the virtual machines in the Greenplum cluster. 

#### <a id="network"></a>network

Configure the network settings for the Greenplum cluster.

```
gpv config set network <setting>
```

Settings:

base-vm <subsetting>
:   Configure the network settings for the base virtual machine. The possible sub-settings are:

        - `gateway-ip <IP>`: Set the gateway Ip address to be used by the base VM when `network-type` is set to `static`. For example: `gpv config set network base-vm gateway-ip 10.0.0.1`
        - `ip <IP>`: Set the static IP address to be used by the base VM when `network-type` is set to `static`. For example: `gpv config set network base-vm ip 10.0.0.5`.
        - `netmask <NETMASK>`: Set the netmask to be used by the base VM to be used by the base VM when network-type is set to `static`. For example: `gpv config set network base-vm netmask 255.255.255.0`.
        - `network-type <TYPE>`: Set the network type tp be used by the base VM. Specifies how the IP addresses are assigned to the base VM. Valid values for `<TYPE>` are `static` and `dhcp`. For example: `gpv config set network base-vm network-type dhcp`.

gp-virtual-etl-bar <subsetting>
:   Configure the network settings for ETL, backup and restore traffic. The possible sub-settings are:
        - `cidr <CIDR>`: Set the CIDR to be used by the `gp-virtual-etl-bar` network. For example: `gpv config set network gp-virtual-etl-bar cidr 192.168.2.1/24`.

gp-virtual-external
:   Configure the network settings of the routable network for Greenplum clients. The possible sub-settings are:
        - `cidr <CIDR>`: Set the CIDR to be used by the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external cidr 10.0.1.1/24`.
        - `dns-servers <DNS server1> <DNS server 2>`: Set the DNS servers of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external dns-servers 8.8.8.8 8.8.4.4`.
        - `gateway-ip <IP>`: Set the gateway IP of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external gateway-ip 10.0.1.1`.
        - `ips <IP1> <IP2>`: Set the static IPs of the `gp-virtual-external` network. If the Greenplum deployment type is `mirrorless`, only one IP address is needed. If the Greenplum deployment type is `mirrored`, two IP addresses are needed. For example: `gpv config set network gp-virtual-external ips 10.0.1.2 10.0.1.3`.
        - `ntp-servers <NTP server1> <NTP server2>`: Set the NTP servers of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external ntp-servers time.example1.com time.example2.com`.

gp-virtual-internal
:   Configure the network settings of the internal communications network among Greenplum virtual machines.
        - `cidr <CIDR>`: Set the CIDR to be used by the `gp-virtual-internal` network. For example: `gpv config set network gp-virtual-internal cidr 192.168.2.1/24`.

#### <a id="vm"></a>VM

Configure individual settings which will appear on all VMs in the cluster. 

```
gpv config set vm <setting>
```

Settings:

base-name <NAME>
:    Set the name of the base VM, which is used during configuring and deploying Greenplum.

gpadmin-password
:    Set the guest operating system password for `gpadmin` user. The utility prompts you to enter a password when you enter this command.

primary-segment-vm-count <NUMBER_OF_SEGMENT_VMS>
:    Set the number of virtual machines to house primary segments. For example: `gpv config set vm primary-segment-vm-count 64`. 

root-password
:    Set the guest operating system password for `root` user. The utility prompts you to enter a password when you enter this command.

#### <a id="vsphere"></a>vSphere

Configure the settings for vSphere (connection parameters, etc).

```
gpv config set vsphere <setting>
```

Settings:

admin-password
:    Set the password to be used for the vCenter administrator. The utility prompts you to enter a password when you enter this command.

admin-username <USER_NAME>
:    Set the username of the vCenter administrator.

compute-cluster-name <COMPUTE_CLUSTER_NAME>
:    Set the name of the compute cluster to use.

datacenter-name <DATACENTER_NAME>
:    Set the name of the datacenter to use.

storage-name <STORAGE_NAME>
:    Set the name of the vSphere datastore or datastore cluster. Specifies the name of the storage layer to use within vCenter. For vSAN, specify the datastore name; for PowerFlex, specify the datastore cluster name.

storage-policy <STORAGE_POLICY>
:    Set the VM storage policy for the storage layer, if applicable. setting specifies the storage policy to be used within vCenter. This setting is required for vSAN only. When specifying the policy, ensure that it is surrounded by quotation marks if the policy's name contains whitespace. For example: `gpv config set vsphere storage-policy "this policy contains spaces"`.

storage-type <STORAGE_TYPE>
:    Set the type of storage layer to use. specifies the type of storage layer to be targeted.  Acceptable values for STORAGE_TYPE include powerflexand vsan.

vcenter-address <VCENTER_ADDRESS>
:    Set the address where the target vCenter may be found. The gpv config set vsphere vcenter-address setting specifies the location of the vCenter on which Greenplum will be deployed.  Note that protocols are not
acceptable in the address (e.g.: http://example.com is invalid because of the prefix http://). For example: `gpv config set vsphere vcenter-address gp.vcenter.example.com`.

vsphere-distributed-switch-mtu <MTU>
:    Specify the MTU for the vSphere distributed switch. setting specifies Maximum Transmission Unit (MTU) of the port groups on the vSphere distributed switch, in bytes. Valid values range from 1500 to 9000.


