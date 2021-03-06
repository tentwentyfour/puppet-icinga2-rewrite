## Example 3 – Using virtual resources and collection in a master-agent set-up.

This is an (almost) complete example for a master-agent set-up using virtual resources in Puppet.

__Note:__ If you're getting "_Error 400 on SERVER: "puppet.domain.tld" is not an Array. It looks to be a String at […]/modules/icinga2/manifests/object/zone.pp:51 on node node.domain.tld_" issues while applying this configuration on your nodes, you're most likely running into a [known bug in the Puppet parser](https://tickets.puppetlabs.com/browse/PUP-1299). In order to make this work, you will need to switch to the [Future Parser](https://docs.puppet.com/puppet/3.8/experiments_future.html):


```
# environment.conf
manifest = site.pp
modulepath = modules:site
parser = future
```

### Manifests

All nodes that should be monitored inherit from the _monitorednode_ role, thus applying the `profile::icinga::agent` class.

Each monitored node exports itself as an endpoint and a zone. This information is then automatically collected on the master to generate the necessary configuration files.

Agents also export a `Host` object (`@@icinga2::object::host`) and use hiera_hash() to get and assemble host properties from the respective hiera files throughout the hiera hierarchy.

#### A word of caution on services and apply rules

This set-up does not use any "manually" created `Service` objects, but _applies_ services to hosts based on their vars exclusively. The result is a much simplier Icinga2 configuration, among other things.

There are two things to note here about `Apply Rules`:

1. We don't use `icinga2::object::service` to define the apply rules, since the current version (0.7.1) of this Puppet module does not yet support the entire range of available functions and macros. Instead, we use `file` resources from a custom, dedicated module, together with the _icinga2::config::file_ tag. The tag makes sure the files will be put in place at the correct point in time and the icinga2 service restarted afterwards (see the module's [README](https://github.com/Icinga/puppet-icinga2-rewrite#custom-configuration) for more information).

An example for one such apply rule file would be:

```
apply Service "nginx-status" to Host {
    import "generic-service"

    vars += host.vars.checks["nginx-status"]
    check_command = "nginx_status"
    command_endpoint = host.vars.client_endpoint

    assign where host.vars.os == "Linux" && host.vars.checks["nginx-status"]
    ignore where !host.address || !host.vars.client_endpoint || !host.vars.checks
}
```

2. Most apply rules are defined on the master and not on the individual nodes. Although it would be preferable to define apply rules as exported resources on the individual nodes – so that they are created only as additional services (profiles) are added to a node, this is not possible when you have more than one node with the same services in your infrastructure since it will lead to duplicate resource definitions.

Compare the `profile::backuppc::server` manifest to the `profile::nginx` profile. In the first, you will find the apply rule definition is exported as a file resource to be collected in `profile::icinga::applyrules` at the very bottom, whereas the second does not contain any `Service` object at all.
Apply rules for the _nginx_ profile have been defined in `profile::icinga::applyrules` since there are several nodes using this same profile.

The nginx profile, however, additionally installs a check script that does not come with any of the _monitoring-plugins-*_ packages on Debian.

### Hiera

With a hiera hierarchy as the following (simplified), all nodes will consume both common.yaml and their dedicated yaml file, if they have one:

```yaml
---
:backends:
  - yaml
:hierarchy:
  - "nodes/%{::fqdn}"
  - common
```

In common.yaml, we define host vars valid for all hosts throughout the infrastructure, while in the respective nodes' yaml files, we define further checks and vars that apply to that host only.

The icingamaster.yaml file contains most of the config necessary for the master configuration.

It's important that we set empty endpoints and zones in this file, so they will not be automatically generated by the icinga2 module, which uses defaults if the values are not set.

```yaml
icinga2::feature::api::endpoints: {}
icinga2::feature::api::zones: {}
```

We use hiera_array() and hiera_hash() lookup functions from the manifests in order to merge arrays and hashes from the various levels in our hierarchy, something that Puppet does not support with automatically looked up values (cf. https://tickets.puppetlabs.com/browse/HI-233)

### Notes

The example is not 100% complete, some of the profile classes that are not relevant to illustrate a master-agent set-up using virtual resources in Puppet are not included and are left as an exercise to the reader.

### Disclaimer

This example does not claim to be a perfect setup. You might have improvements to suggest and those are of course welcome.
