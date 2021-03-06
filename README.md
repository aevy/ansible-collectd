collectd role
===============

This role takes care of adding collectd to any given server.

It can be used to:

1. Collect metrics about a machines, and optionally forward them to an aggregation machine.

2. Create an aggregation machine, which will receive the metrics from a given set of machine within the same datacenter.
   In this case too, metrics can optionally be forwarded to another service.

Configuration
--------------

### For any machines

Simply add the collector role to your playbook.

By default, we will save all metrics to RRD files under `/var/lib/collectd/rrd/${hostname}`.

### For forwarding machines

#### Inventory

Normally, we put all aggregation on a single machine.
However, it is possible to decouple them, and provision separate machines for any aggregators we might have.
This example demonstrate how to create a dedicated collectd aggregator.

```ini
[collectd:children]
collectd-somedc-prod

[collectd-somedc-prod]
collectd1.somedc.prod         ansible_ssh_host=10.0.1.111

[... skipping ...]
[somedc-prod:children]
collectd-somedc-prod

[somedc-prod:vars]
collectd_forwarder = collectd1.somedc.prod

#
# Optional: if you are aggregating logs with logstash
# and are using ElasticSearch and Kibana, you might
# want to also keep track of your metrics through
# Kibana. If so, simply add the following to your
# global configuration - keep in mind that
# activating this will turn off any other
# forwarding you might have configured in
# favor of sending all entries to logstash
#
logstash_forwarder = collectd2.somedc.prod

collectd_forward_to_logstash = true
```

The following may also be added to your inventory.

* `collectd_interval`: at what interval in seconds to take measurements (default: 60)
* `check_disk`: a value which we will use to select what disk to monitor (default: xvde)
* `collectd_df_fs_type`: the file system type to monitor (Collectd 5.4 default: ext4. Collectd 5.5 default: undefined)
* `collectd_df_device`: the dev device to monitor.
* `collectd_df_mountpoint`: the mount point to monitor.
* `collectd_df_IgnoreSelected`: [see wiki](https://collectd.org/wiki/index.php/Plugin:DF#Parameters) (default: false)
* `collectd_df_ReportByDevice`: [see wiki](https://collectd.org/wiki/index.php/Plugin:DF#Parameters) (default: false)
* `collectd_df_ReportInodes`: [see wiki](https://collectd.org/wiki/index.php/Plugin:DF#Parameters) (default: false)
* `collectd_df_ValuesAbsolute`: [see wiki](https://collectd.org/wiki/index.php/Plugin:DF#Parameters) (default: true)
* `collectd_df_ValuesPercentage`: [see wiki](https://collectd.org/wiki/index.php/Plugin:DF#Parameters) (default: false)
* `monitor_coretemp`: set to true if you want to monitor coretemp (only useful on real hardware)
* `private_ip`: set to the IP address of the `collectd_forwarder` if `collectd_fowarder` has multiple IPs. Example:
```
collectd1.somedc.prod         ansible_ssh_host=10.0.1.111   private_ip=172.16.1.111
```

### Plugin configuration

The `collectd_plugins` variable defines the plugins to be loaded. Default plugins are below:

```
  - syslog
  - cpu
  - interface
  - load
  - memory
  - vmem
  - swap
  - disk
  - rrdtool
  - network
  - librato
  - exec
  - write_graphite
```
  

#### [Disk](https://collectd.org/wiki/index.php/Plugin:Disk)


Default: xvde

```
check_disk: xvde
```

#### [DF](https://collectd.org/wiki/index.php/Plugin:DF)

All options are optional.

The following should be set as an array.
Example 1: (Single value)
```
collectd_df_device: '["/dev/xvda1"]'
collectd_df_mountpoint: '["/"]'
collectd_df_fs_type: '["xfs"]'
```
Example 2: (Multiple values)
```
collectd_df_device: '["/dev/xvda1","/dev/xvdb1"]'
collectd_df_mountpoint: '["/","/data"]'
collectd_df_fs_type: '["xfs","ext4"]'
```

You can change the rest of the defaults.
```
collectd_df_IgnoreSelected: false
collectd_df_ReportByDevice: false
collectd_df_ReportInodes: false
collectd_df_ValuesAbsolute: true
collectd_df_ValuesPercentage: false
```

### Install plugins

To indicate the plugins that you would like installed, list all the plugins in the `packages` variable in 
your `group_vars` folder.

For example, below installs only the `collect-rrdtools`, `collectd-rrdtool` and `collectd-iptables` plugins.

```
packages:
  - collectd-rrdcached
  - collectd-rrdtool
  - collectd-iptables
```

If you do not set `packages`, the following default packages that will be installed.

```
  - git
  - collectd
  - collectd-amqp
  - collectd-apache
  - collectd-bind
  - collectd-curl
  - collectd-curl_json
  - collectd-curl_xml
  - collectd-dbi
  - collectd-dns
  - collectd-generic-jmx
  - collectd-iptables
  - collectd-ipvs
  - collectd-java
  - collectd-lvm
  - collectd-mysql
  - collectd-netlink
  - collectd-notify_desktop
  - collectd-postgresql
  - collectd-rrdcached
  - collectd-rrdtool
  - collectd-sensors
  - collectd-snmp
  - collectd-write_riemann
```

The following command will list the possible packages that can be installed:

  - CentOS 6: `sudo yum --enablerepo="epel,gf-plus" list available | grep collectd-`
  - CentOS 7: `sudo yum --enablerepo="epel" list available | grep collectd-`


### Roles addition

It is possible for any roles to add their own custom metric collection configuration.

In `myrole/templates/collectd.conf.j2`:

```
<Plugin "whatever">
    ...
</Plugin>
```

Then, in `myrole/tasks/main.yml`:

```yaml
- name: Adding collectd monitoring for myrole
  template: >
    src=collectd.conf.j2
    dest=/etc/collectd.d/myrole.conf
  notify:
    - Restart collectd
  tags:
    - myrole
    - files
    - collectd
```

### Execution

#### For any machines

Simply follow the standard provisioning method.

### For the collector machine

When provisioning with this role, you can also add
[Librato](https://www.librato.com/) and
[Graphite](http://graphite.wikidot.com/) support.

As you should not add any credentials to your inventory,
it should be done by using `--extra-vars` as described below.

#### Librato 

```
--extra-vars="use_librato=true" \
--extra-vars="librato_email=some@email.com" \
--extra-vars="librato_token=API-KEY"
```

### Graphite

```
--extra-vars="use_graphite=true" \
--extra-vars="graphite_host=some.host.com" \
--extra-vars="graphite_port="
```

See also
--------

* [collectd](http://collectd.org/)
