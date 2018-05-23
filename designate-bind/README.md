# Integration of Designate backend and BIND front end deployed with openstack-ansible

This describes deploying the designate back-end with openstack-ansible, and creating public facing BIND servers on known fixed IP.

## Create container config

Copy env.d/dnsaas-bind.yml to /etc/openstack-deploy to define the BIND service layout

Define desigate back end hosts in openstack_user_config.yml

```yaml
# designate
dnsaas_hosts:
  infra1: *_infra1_
  infra2: *_infra2_
  infra3: *_infra3_
```

Define BIND front end container hosts in openstack_user_config.yml. These must have a bridge carrying a public network.

```yaml
# bind servers for designate
dnsaas-bind_hosts:
  haproxy1: *_haproxy1_
  haproxy2: *_haproxy2_
```

## Copy designate and BIND server config into place

Copy user_variables_designate.yml and designate_pools.j2 to /etc/openstack-deploy to define the pool of BIND servers that designate will use
[ there must be a nicer way of doing this? ]

Copy groupvars/dnsaas-bind.yml to /etc/openstack-deploy/group_vars to configure the specific networking for the public BIND servers
Edit network setup to suit.
[ this is a hack as the master version of lxc_containers_create finally has sufficient vars that can be overridden to allow a specific interface config to be side-loaded per container. The dynamic inventory cannot do this ]

## Create the containers

```
cd /opt/openstack-ansible
openstack-ansible containers-lxc-create.yml --limit designate_all
openstack-ansible playbooks/containers-lxc-create.yml --limit dnsaas-bind_all
```

## Set up the HAProxy services

```
openstack-ansible haproxy-install.yml --tags=haproxy-service-config
```

## Deploy the back end designate services

```
openstack-ansible playbooks/os-designate-install.yml
```

## Deploy the front end bind services

```
cd /opt/openstack-ansible-ops/designate-bind
openstack-ansible installBind.yml
```

## TODO:

* Designate HA configuration looks wrong - no coordinator?
* rncd is missing from designate containers when BIND is not co-installed, this needs to be [always|conditionally] installed
* No way to define / share / install the rncd key into the designate container - is this a user_secret or coming from somewhere else?
