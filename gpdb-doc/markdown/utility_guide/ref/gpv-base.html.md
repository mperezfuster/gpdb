# gpv base

Manages the base virtual machine for a deployment.
The `gpv base` command group allows you to import and deploy the base virtual machine.

## <a id="section2"></a>Usage

```
gpv base <command>
```

## <a id="opts"></a>Commands

### <a id="deploy"></a>Deploy

Configure the imported base VM for Greenplum deployment. The `gpv base deploy` command performs the following operations in the base vm:

- Copy and install Greenplum and Greenplum Virtual Service packages.
- Configure the OS using the Greenplum Virtual Service.
- Change secrets for `root` and `gpadmin` roles.
- Validate that the base virtual machine is properly configured.

### <a id="import"></a>Import

Import an existing VM template or an OVA file as a new base VM. The `gpv base import` command creates a new base virtual machine. This virtual machine can be created from an existing virtual machine template provided by the user, usually already certified to follow all the existing policies. It may also be created from an OVA file. For example, the VMWare provided OVA with all the dependencies required for Greenplum deployment in an air-gaped environment.

Options:

-f, --ova=file_name
:   Specify a local OVA file name to be uploaded to the target vSphere cluster.

-t, --template=template_name
:   Specify an existing virtual machine template `template_name` to be cloned.


## <a id="examples"></a>Examples

Import an existing VM template:

```
gpv base import -t customer-provided-template
```

Import a local OVA:

```
gpv base import -f /path/to/pre-built.ova
```


