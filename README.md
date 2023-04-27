# puppet-keepalived

## Overview

Install, enable and configure the keepalived VRRP and LVS management daemon.

* `keepalived` : Main class to install, enable and configure the service.
* `keepalived::vrrp` : Wrapper class for VRRP-only keepalived setups.

The configuration file to be used can be specificed using either the `$content`
parameter (typically for templates), or the `$source` parameter. If neither is
specified, you will need to manage it from elsewhere.

See the `templates/sysconfig.erb` file for the possible `$options` values. The
default is `-D` which enables both VRRP and LVS.

## Examples

Typical installation for VRRP only (no LVS), using a static existing
configuration file :

```puppet
class { '::keepalived':
  source  => "puppet:///${module_name}/keepalived.conf",
  options => '-D --vrrp',
}
```

Similar to the above, but using a template, which can be useful with multiple
servers which will be part of the same VRRP group and/or have the same LVS
configuration :

```puppet
class { '::keepalived':
  content => template("${module_name}/keepalived.conf.erb"),
}
```

For the `keepalived::vrrp` class, configuration for the template needs to be
passed into the `$global_defs` and `$instances` hash parameters. Their
structure follows the structure of the `keepalived.conf` file :

```puppet
case $::hostname {
  'web1':  { $vrrp_state = 'MASTER' $vrrp_priority = '50' }
  default: { $vrrp_state = 'BACKUP' $vrrp_priority = '10' }
}
class { '::keepalived::vrrp':
  global_defs => {
    router_id => $::hostname,
  },
  instances => {
    web => {
      advert_int        => '3',
      authentication    => {
        auth_type => 'PASS',
        auth_pass => 'abcd1234',
      },
      interface         => 'eth1',
      priority          => $vrrp_priority,
      state             => $vrrp_state,
      virtual_ipaddress => {
        '10.0.0.13/24' => 'dev eth0 label eth0:13',
        '10.0.1.13/24' => 'dev eth1 label eth1:13',
      },
      virtual_router_id => '13',
    },
  },
}
```
The same but using track_script :
```puppet
case $::hostname {
  'web1':  { $vrrp_state = 'MASTER' $vrrp_priority = '50' }
  default: { $vrrp_state = 'BACKUP' $vrrp_priority = '10' }
}
class { '::keepalived::vrrp':
  global_defs => {
    router_id => $::hostname,
    debug_script => 'all',
  },
  instances => {
    web => {
      advert_int        => '3',
      authentication    => {
        auth_type => 'PASS',
        auth_pass => 'abcd1234',
      },
      interface         => 'eth1',
      priority          => $vrrp_priority,
      state             => $vrrp_state,
      virtual_ipaddress => {
        '10.0.0.13/24' => 'dev eth0 label eth0:13',
        '10.0.1.13/24' => 'dev eth1 label eth1:13',
      },
      virtual_router_id => '13',
      notify_master => "/path/to/script.sh",
      track_script => {
        haproxy_check => {
          script => "/usr/libexec/haproxy_check.sh",
          interval => 2,
          weight => 2,
          rise => 2,
          fall => 2,
        },
        nginx_check => {
          script => "/usr/libexec/nginx_check.sh",
          interval => 3,
          weight => 3,
          rise => 3,
          fall => 3,
        },
      },
    },
  },
}
```

Note that you may also set the `$global_defs_defaults` parameter, which will
be merged with the more specific `$global_defs`, which is especially useful
with hiera :

```yaml
---
keepalived::vrrp::global_defs_defaults:
  notification_email:
    - 'root@example.com'
  notification_email_from: 'root@example.com'
  router_id: "%{::hostname}"
  smtp_connect_timeout: '30'
  smtp_server: '10.0.45.156'
```

