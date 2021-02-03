---
title: "Puppet 4 + Hiera 5 + Hashicorp Vault"
date: 2020-10-09T23:15:54Z
draft: false
---

Quick guide for setting up Vault with Puppet

## What doesn't this cover
* Building a Production grade Vault cluster
* Puppet 5 and 6 (Yes, I know Puppet 4 is EOL)
* Vault Dynamic Secrets
* The more secure [Puppet 6 deferred functions](https://www.hashicorp.com/resources/agent-side-lookups-with-hashicorp-vault-puppet-6)

## Setup Vault

Since I have a Kubernetes cluster, I used the Vault Helm Chart to set this up, I mostly followed this guide to setup Vault in Dev mode, which looks to be easy to then convert to a real Raft based Vault cluster to productionise it afterwards: https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide

Vault should be accessible outside of your Cluster, I configured mine to use the ingress controller.

As half of my infra is Puppet and more an more being moved into K8s, I benefit from being able to share secrets between the too using the [Vault Agent Sidecar Injector](https://www.vaultproject.io/docs/platform/k8s/injector)

## Install dependencies
```
# /opt/puppetlabs/puppet/bin/gem install vault
# /opt/puppetlabs/puppet/bin/gem install debouncer
# puppetserver gem install vault
# puppetserver gem install debouncer
```

## Install hiera_vault
This is the module/function which Hiera uses. I use r10k to manage my Puppet modules so I added this to my Puppetfile in the repo
```
mod 'petems/hiera_vault', '1.0.0'
```

## Hiera diff for the config I needed

```diff
---
version: 5
defaults:
  datadir: hieradata
  data_hash: yaml_data

hierarchy:
+  - name: "Hiera-vault lookup"
+    lookup_key: hiera_vault
+    options:
+      confine_to_keys:
+        - '^vault_.*'
+        - '^.*_password$'
+        - '^password.*'
+      ssl_verify: false
+      address: http://vault.example.net:80
+      token: root
+      default_field: value
+      mounts:
+        puppet:
+          - "%{::trusted.certname}"
+          - "common"
+

  - name: "Data"
    paths:
      - "nodes/%{::trusted.certname}"
      - "%{fqdn}"
      - "common.yaml"
```

## Create a space in Vault for us to keep our secrets in:
```
$ vault secrets enable -version=2 -path="puppet" kv
Success! Enabled the kv secrets engine at: puppet/
```

## Add a value to Vault
```
$ vault kv put puppet/common/vault_secretxyz value='Default Secret'
Key              Value
---              -----
created_time     2020-10-09T10:50:00.208756081Z
deletion_time    n/a
destroyed        false
version          1

$ vault kv put puppet/server1.example.net/vault_secretxyz value='server1.example.net secret'
Key              Value
---              -----
created_time     2020-10-09T10:50:22.734782394Z
deletion_time    n/a
destroyed        false
version          1
```

## Testing values
### Server that matches value
Now we can use the `puppet lookup` command to check these values
```
$ puppet lookup vault_secretxyz --node=server1.example.net
--- server1.example.net secret
...
```

### Default Value
```
$ puppet lookup vault_secretxyz --node=server2.example.net
--- Default Secret
...
```

## Notes
### Variables
Using variables in your hiera like `%{::trusted.certname}` can be confusing. If the node name doesn't exist it won't relate to the facts that Puppet has so the `certname` will be `null`. Lost me for a little while.

### Protecting your Vault key
For production, best not to have your vault key in public. It's possible for hiera to pick up an env var, `VAULT_TOKEN`. This probably needs setting in `/etc/default/puppetserver`. https://github.com/petems/petems-hiera_vault/issues/35

### Puppet 4
Reading the docs took me down a rabbithole of needing jruby-9k. This isn't possible for Puppet 4 (it is in Puppet 5 and then default in Puppet 6). I burnt an hours trying to figure this out.


## Resources
List of resources which got me sorted:

* https://github.com/petems/petems-hiera_vault
* https://github.com/petems/puppet-hiera-vault-vagrant
* https://petersouter.xyz/how-to-use-vault-with-hiera-5-for-secret-management-with-puppet/
* https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide
