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

Where `setting` can be one of the following:

deployment-type <type_name>
:   Specifies whether the Greenplum deployment uses mirror segments or not. Valid values of `<type_name>` include `mirrored` and `mirrorless`. For example: `gpv config set database deployment-type mirrored`. 

prefix <prefix_name>
:   Specifies a label which serves as a prefix for the names of the resource pool and the virtual machines in the Greenplum cluster. 

#### <a id="network"></a>network

Configure the network settings for the Greenplum cluster.

```
gpv config set network <setting>
```

Where `setting` can be one of the following:

base-vm <sub-setting>
:   Configure the network settings for the base virtual machine. The possible sub-settings are:
    - `gateway-ip <IP>`: Set the gateway IP address for the base virtual machine when `network-type` is set to `static`. For example: `gpv config set network base-vm gateway-ip 10.0.0.1`
    - `ip <IP>`: Set the static IP address for the base virtual machine when `network-type` is set to `static`. For example: `gpv config set network base-vm ip 10.0.0.5`.
    - `netmask <NETMASK>`: Set the netmask for the base virtual machine when network-type is set to `static`. For example: `gpv config set network base-vm netmask 255.255.255.0`.
    - `network-type <TYPE>`: Set the network type for base virtual machine. The possible values are `static` and `dhcp`. For example: `gpv config set network base-vm network-type dhcp`.

gp-virtual-etl-bar <sub-subsetting>
:   Configure the network settings for ETL, backup and restore traffic. The possible sub-settings are:
    - `cidr <CIDR>`: Set the Classless Inter-Domain Routing (CIDR) for the `gp-virtual-etl-bar` network. For example: `gpv config set network gp-virtual-etl-bar cidr 192.168.2.1/24`.

gp-virtual-external <sub-subsetting>
:   Configure the network settings of the routable network for Greenplum clients. The possible sub-settings are:
    - `cidr <CIDR>`: Set the CIDR for the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external cidr 10.0.1.1/24`.
    - `dns-servers <DNS server1> <DNS server 2>`: Set the DNS servers of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external dns-servers 8.8.8.8 8.8.4.4`.
    - `gateway-ip <IP>`: Set the gateway IP of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external gateway-ip 10.0.1.1`.
    - `ips <IP1> <IP2>`: Set the static IPs of the `gp-virtual-external` network. If the Greenplum deployment type is `mirrorless`, you only need one IP address. If the Greenplum deployment type is `mirrored`, you need two IP addresses. For example: `gpv config set network gp-virtual-external ips 10.0.1.2 10.0.1.3`.
    - `ntp-servers <NTP server1> <NTP server2>`: Set the NTP servers of the `gp-virtual-external` network. For example: `gpv config set network gp-virtual-external ntp-servers time.example1.com time.example2.com`.

gp-virtual-internal <sub-setting>
:   Configure the network settings of the internal communications network among Greenplum virtual machines. The possible sub-settings are:
    - `cidr <CIDR>`: Set the CIDR to be used by the `gp-virtual-internal` network. For example: `gpv config set network gp-virtual-internal cidr 192.168.2.1/24`.

#### <a id="vm"></a>vm

Configure individual settings for all virtual machines in the cluster. 

```
gpv config set vm <setting>
```

Where `setting` can be one of the following:

base-name <base-VM-name>
:    Set the name of the base virtual machine to use during configuring and deploying Greenplum on vSphere.

gpadmin-password
:    Set the guest operating system password for `gpadmin` user. The utility prompts you to enter a password when you enter this command.

primary-segment-vm-count <number-of-segment-vms>
:    Set the number of virtual machines to house primary segments. For example: `gpv config set vm primary-segment-vm-count 64`. 

root-password
:    Set the guest operating system password for `root` user. The utility prompts you to enter a password when you enter this command.

#### <a id="vsphere"></a>vsphere

Configure the settings for vSphere.

```
gpv config set vsphere <setting>
```

Where `setting` can be one of the following:

admin-password
:    Set the password for the vCenter administrator. The utility prompts you to enter a password when you enter this command.

admin-username <user-name>
:    Set the username of the vCenter administrator.

compute-cluster-name <compute-cluster-name>
:    Set the name of the compute cluster to use.

datacenter-name <datacenter-name>
:    Set the name of the datacenter to use.

storage-name <storage-name>
:    Set the name of the vSphere datastore or datastore cluster. Specify the name of the storage layer to use within vCenter. For vSAN, specify the datastore name; for Dell PowerFlex, specify the datastore cluster name.

storage-policy <storage-policy>
:    Set the virtual machine storage policy for the storage layer, if applicable. This setting is required for vSAN only. When specifying the policy, ensure that it is surrounded by quotation marks if the policy's name contains whitespace. For example: `gpv config set vsphere storage-policy "this policy contains spaces"`.

storage-type <storage-type>
:    Set the type of storage layer to use. Possible values are `powerflex` and `vsan`.

vcenter-address <vcenter-address>
:    Set the address of the target vCenter. Do not include the protocol prefix in the address (http://, https://). For example: `gpv config set vsphere vcenter-address gp.vcenter.example.com`.

vsphere-distributed-switch-mtu <mtu>
:    Specify the Maximum Transfer Unit (MTU) for the vSphere distributed switch, in bytes. Valid values range from 1500 to 9000.
