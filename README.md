This Puppet module installs and configures an EPICS soft IOC. The IOC is run in
`procServ` to ease maintenance. The IOC can be started/stopped as a system
service.
On systems that use systemd as init system (e.g. Debian >=8) a systemd unit
file will be created for each IOC process. On older systems (e.g. Debian 7) this
module relies on sysv-rc-softioc for creating System V init scripts instead.

# Environment Variables

Some attributes of the `Epics_softioc::Ioc` class cause environment variables to
be set. Please refer to the following table for a list:

| Attribute            | Environment Variable       |
|----------------------|----------------------------|
| `ca_addr_list`       | `EPICS_CA_ADDR_LIST`       |
| `ca_auto_addr_list`  | `EPICS_CA_AUTO_ADDR_LIST`  |
| `ca_max_array_bytes` | `EPICS_CA_MAX_ARRAY_BYTES` |
| `log_port`           | `EPICS_IOC_LOG_PORT`       |
| `log_server`         | `EPICS_IOC_LOG_INET`       |
| `ca_sec_file`        | `EPICS_CA_SEC_FILE`        |

Environment variables that are not on this list can be set using the `env_vars`
attribute.

# Examples

## Simple

```
  $iocbase = '/usr/local/lib/iocapps'

  class { 'epics_softioc':
    iocbase => $iocbase,
  }

  epics_softioc::ioc { 'mysoftioc':
    ensure  => running,
    enable  => true,
    bootdir => 'iocBoot/ioclinux-x86_64',
    require => File["${iocbase}/mysoftioc"],
  }
```

## Complex

In this example we run two IOC processes on the same machine. Thus different
`procServ` ports need to be used. See the first block for an example of setting
facility-wide defaults.

```
Epics_softioc::Ioc {
  log_port            => 6500,
  log_server          => 'logserver.example.com',
  manage_autosave_dir => true,
}

$iocbase = '/usr/local/lib/iocapps'

class { 'epics_softioc':
  iocbase => $iocbase,
}

epics_softioc::ioc { 'vacuum':
  ensure            => running,
  enable            => true,
  bootdir           => 'iocBoot/ioclinux-x86_64',
  console_port      => 4051,
  ca_addr_list      => '',
  ca_auto_addr_list => true,
  env_vars          => {
    'EPICS_CA_CONN_TMO'     => '1',
    'EPICS_CA_NAME_SERVERS' => 'nameserver.example.com',
  },
  uid               => 900,
  run_make          => false,
  require           => File["${iocbase}/vacuum"],
}

epics_softioc::ioc { 'llrf':
  ensure       => running,
  enable       => true,
  bootdir      => 'iocBoot/ioclinux-x86_64',
  console_port => 4052,
  uid          => 901,
  require      => File["${iocbase}/llrf"],
}
```

# Reference

## Class `epics_softioc`

This class takes care of all global tasks which are needed in order to run a
soft IOC. It installs the needed packages and prepares machine-global
directories and configuration files.

### `gid`

Define the group id of the group the IOCs are run as. This group is used for
granting access to the common log directory. The gid will be picked
automatically if this option is not specified.

### `iocbase`

By default the IOC directories are expected to be all in one location. This
option specifies the path to this base directory. Default:
`/usr/local/lib/iocapps`.

## Defined Type `epics_softioc::ioc`

This type configures an EPICS soft IOC. It creates configuration files,
automatically populates them with the correct values and configures the IOC for
running it as a system service. The top-level IOC directory of the IOC is
expected to be `$iocbase/<ioc_name>` where <ioc_name> is the name specified for
the `epics_softioc::ioc` resource.

### `abstopdir`

Absolute path to the IOC's top-level directory. Defaults to
`<iocbase>/<ioc_name>`.

### `auto_restart_ioc`

Whether to restart the IOC after recompiling. If set to `true` the IOC will be
restarted automatically after recompiling the source code (see `run_make`). This
ensures the latest code is being used. The default is `true`.

### `autosave_base_dir`

The path to the base directory for the EPICS `autosave` module. Defaults to
`/var/lib`.

### `bootdir`

Path to the directory containing the IOC start script. This path is relative to
`abstopdir`. This defaults to `iocBoot/ioc\${HOST_ARCH}`.

### `ca_addr_list`

Allows to configure the `EPICS_CA_ADDR_LIST` environment variable for the IOC.
The default is undefined (environment variable not set).

### `ca_auto_addr_list`

Allows to configure the `EPICS_CA_AUTO_ADDR_LIST` environment variable for the
IOC. Valid values are `true` and `false`. The default is undefined (environment
variable not set).

### `ca_max_array_bytes`

Allows to configure the `EPICS_CA_MAX_ARRAY_BYTES` environment variable for the
IOC. The default is undefined (environment variable not set).

### `cfg_append`

Allows to set additional variables in the IOC's config file in `/etc/iocs/`.
This is not used on machines that use systemd.

### `console_port`

Specify the port number `procServ` will listen on for connections to the IOC
shell. You can connect to the IOC shell using `telnet localhost <portnumber>`.
The default port is `4051`.

Note that access is not possible if `enable_console_port` is set to `false`.

### `coresize`

The maximum size (in Bytes) of a core file that will be written in case the IOC
crashes. The default is `10000000`.

### `enable_console_port`

If set to `true` (the default) `procServ` will listen on the port specified by
`console_port` for connections to the IOC shell. If this flag is `true` for at
least one IOC `telnet` is being installed.

### `enable_unix_domain_socket`

If set to `true` (the default) `procServ` will create a unix domain socket for
connections to the IOC shell. If this flag is `true` for at least one IOC the
BSD version of `netcat` is installed.

### `ensure`

Ensures the IOC service is running/stopped. Valid values are `running`,
`stopped`, and <undefined>. If not specified Puppet will not start/stop the IOC
service.

### `enable`

Whether the IOC service should be enabled to start at boot. Valid values are
`true`, `false`, and <undefined>. If not specified (undefined) Puppet will not
start/stop the IOC service.

### `env_vars`

Specify a hash of environment variables that shall be passed to the IOC.
Defaults to `{}`.

### `log_port`

Allows to configure the `EPICS_IOC_LOG_PORT` environment variable for the IOC.
The default is 7004 (the default port used by iocLogServer).

### `log_server`

Allows to configure the `EPICS_IOC_LOG_INET` environment variable for the IOC.
The default is undefined (environment variable not set).

### `ca_sec_file`

Allows to configure the `EPICS_CA_SEC_FILE` environment variable for the IOC.
The default is undefined (environment variable not set).
To be used as input for `asSetFilename(${EPICS_CA_SEC_FILE})`.

### `logrotate_compress`

Whether to compress the IOC's log files when rotating them. Defaults to `true`.

### `logrotate_rotate`

The time in days after which a the log file for the procServ log will be
rotated. Default is `30`.

### `logrotate_size`

If the log file for the procServ log reaches this size the IOC log will be
rotated. Default is `'10M'`.

### `manage_autosave_dir`

Whether to automatically populate the `AUTOSAVE_DIR` environment variable.
Valid values are `true` and `false`. If true the specified directory will be
created (users need to ensure the parent directory exists) and permissions will
be set up appropriately. The `AUTOSAVE_DIR` environment variable will be set to
<autosave_base_dir>/softioc-<ioc_name>. Also see `autosave_base_dir`.

### `manage_user`

Whether to create the user account the IOC is running as. Set to false to use a
user account that is managed by Puppet code outside of this module. Disable if
you want multiple IOCs to share the same user account. Defaults to `true`.

### `procserv_log_file`

The log file that `procServ` uses to log activity on the IOC shell. Default:
`/var/log/softioc-<ioc_name>/procServ.log`.

### `run_make`

Whether to compile the IOC when its source code changes. If set to `true` the
code in the IOC directory will be compiled automatically by running `make`. This
ensures the IOC executable is up to date. The default is `true`.

Note: This module runs `make --question` to determine whether it needs to
rebuild the code by running make. Some Makefiles run a command on every
invocation. This can cause `make --question` to always return a non-zero exit
code. Beware that this will cause Puppet to rebuild your IOC on every run.
Depending on the `auto_restart_ioc` setting this might also cause the IOC to
restart on every Puppet run! Please verify that your Makefiles are behaving
correctly to prevent surprises.

### `run_make_after_pkg_update`

If this option is activated the IOC will be recompiled whenever a `package`
resource tagged as `epics_ioc_pkg` is refreshed. This can be used to rebuild
IOCs when facility-wide installed EPICS modules like autosave are being
updated. The default is `true`.

### `unix_domain_socket`

Specify the Unix domain socket file `procServ` will create for connections to
the IOC shell. The file name has to be specified relative to the run-time
directory (`/run`). You can connect to the IOC shell using
`nc -U <unix_domain_socket>`. The default is `softioc-<ioc_name>/procServ.sock`.

Note that the unix domain socket will not be created if
`enable_unix_domain_socket` is set to `false`.

#### Example

Facility-wide IOC profile:
```
  class { 'epics_softioc':
    iocbase => '/usr/local/lib/iocapps',
  }

  package { 'epics-autosave-dev':
    ensure => latest,
    tag    => 'epics_ioc_pkg',
  }
```

Machine-specific manifest:
```
  epics_softioc::ioc { 'mysoftioc1':
    ...
  }

  epics_softioc::ioc { 'mysoftioc2':
    ...
    run_make_after_pkg_update => false,
  }
```

In this example `mysoftioc1` will automatically be recompiled (and restarted
if `auto_restart_ioc` is `true`) when Puppet installs a new version of the
`epics-autosave-dev` package.

### `startscript`

Base file name of the IOC start script. This defaults to `st.cmd`.

### `systemd_after`

Ensures the IOC service is started after the specified `systemd` units have been
activated. Please specify an array of strings. Default: `network.target`.

This parameter is ignored on systems that are not using `systemd`.

Note: This enforces only the correct order. It does not cause the specified
targets to be activated. Also see `systemd_requires`.

### `systemd_requires`

Ensures the specified `systemd` units are activated when this IOC is started.
Default: `network.target`.

This parameter is ignored on systems that are not using `systemd`.

Note: This only ensures that the required services are started. That generally
means that `systemd` starts them in parallel to the IOC service. Please use
`systemd_after` to ensure they are started before the IOC is started.

### `systemd_requires_mounts_for`

Ensures the specified paths are accessible (e.g. the corresponding file systems
are mounted) when this IOC is started. Specify an array of strings. Default:
`[]`.

This parameter is ignored on systems that are not using `systemd`.

### `uid`

Defines the system user id the IOC process is supposed to run with. The
corresponding user is created automatically. If you leave this undefined an
arbitrary user id will be picked. This argument is only used if `manage_user` is
`true`.

### `username`

The user name the IOC will run as. By default `softioc-<ioc_name>` is being
used.

# Contact

Author: Martin Konrad <konrad at frib.msu.edu>
