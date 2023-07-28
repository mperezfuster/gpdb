# gppkg 

Installs Greenplum Database extensions in `.gppkg` format, such as PL/Java, PL/R, PostGIS, and MADlib, along with their dependencies, across an entire cluster.

## <a id="synopsis"></a>Synopsis 

```
gppkg <command> [<command_options> ...] 

gppkg <commmand> --h | --help

gppkg --version

gppkg -v | --verbose
```

## <a id="description"></a>Description 

The Greenplum Package Manager -- `gppkg` -- utility installs Greenplum Database extensions, along with any dependencies, on all hosts across a cluster. It will also automatically install extensions on new hosts in the case of system expansion and segment recovery.

The `gppkg` utility does not require that Greenplum Database is running in order to install packages.

> **Note** After a major upgrade to Greenplum Database, you must download and install all `gppkg` extensions again.

Examples of database extensions and packages software that are delivered using the Greenplum Package Manager:

-   PL/Java
-   PL/R
-   PostGIS
-   MADlib

## <a id="commands"></a>Commands

help 
:   Displays the help for the command.

install [--tmpdir] <package-name>
:   Installs the specified package in the cluster. This includes any pre/post installation steps and installation of any dependencies.

migrate [options] --source <SOURCE> --destination <DESTINATION> 

:   Migrates packages from a separate $GPHOME. Carries over packages from one version of Greenplum Database to another.

For example: `gppkg --migrate /usr/local/greenplum-db-<old-version> /usr/local/greenplum-db-<new-version>`.

> **Note** In general, it is best to avoid using the --migrate option, you should reinstall packages instead of migrating them.

query
:   Displays which extensions are installed in the cluster.

remove <package-name>
:    Uninstalls the specified package from the cluster. 

upgrade
:    Upgrades an existing package. Does a fresh install if the package does not exist.
gppkg upgrade pkg-name.gppkg

## <a id="options"></a>Options 

-a | --accept 
:   Do not prompt the user for confirmation.

-d | --dryrun     
:   Run a simulation for the command, without modifying anything.

-f | --force
:   Skip all requirement checks and overwrite existing files.

-h | --help
:   Display the online help.

-V | --version
:   Displays the version of this utility.

-v | --verbose
:   Sets the logging level to verbose.

