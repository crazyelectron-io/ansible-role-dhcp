# ansible-role-dhcp

Ansible role to setup a high available master-slave DHCP configuration using `isc-dhcp-server` (tested on Debian 12) as companion to `BIND9`.

> Be aware that this role is opinionated and fits my preferences and way of working.
> It may or may not be suitable for your needs.

## Role variables

### Mandatory variables

`dhcp_interface:` must be specified to bind the DHCP server to the correct interface to listen for DHCP requests.

`search_domain` must be defined to allow the DHCP server to add the domain name to the hostename.

`primary_dns` must be defined to pass that on the clients.

These variables can be specified in the `all.sops.yaml` inventory file (prefered) or in the `vars`section of an Ansible playbook, like this:

```yaml
- hosts: all
  become: true
  gather_facts: true
  vars:
    dhcp_interface: bond0
    search_domain: example.com

  roles:
    ...
    - dhcp
  ...
```

### Important optional parameters

`dhcp_reservations` can optionally specify reserved IP addresses (based on the device MAC address) by adding them to the group varaiables file:

```yaml
dhcp_reservations:
  - host: hue01
    mac: 00:17:88:2f:cd:a3
    address: 10.20.2.123
  - host: airco-bedroom
    mac: 00:23:11:ef:ca:7e
    address: 10.20.1.123
```

A`dhcp_hostname_overrides` can optionally be used to fix invalid hostnames (e.g. with underscores like some ESP devices):

```yaml
dhcp_hostname_overrides:
  - host: ESP-12345
    mac: 00:17:88:2f:cd:a4
    name: esp-12345
```

`secondary_dns` - adds a secondary DNS server (in addition to the primary DNS server).

`default_lease_time` - defaults to 14400 seconds (4 hours) - can be overwritten with whatever is appropriate for the local network.

## Create subnets

An example configuration for DHCP subnet configuration is included and can be used as starting point for your own subnet configuration:

```yaml
subnet 10.0.0.0 netmask 255.255.255.0 {
    option broadcast-address    10.0.0.255;
    option subnet-mask          255.255.255.0;
    option routers              10.0.0.1;
    pool {
        failover peer "dhcp-failover";
        range 10.0.0.100 10.0.0.200;
    }
    if exists user-class and ( option user-class = "iPXE" ) {
        filename "http://boot.netboot.xyz/menu.ipxe";
    } elsif option arch = encode-int ( 16, 16 ) {
        filename "http://boot.netboot.xyz/ipxe/netboot.xyz.efi";
        option vendor-class-identifier "HTTPClient";
    } elsif option arch = 00:07 {
        filename "netboot.xyz.efi";
    } else {
        filename "netboot.xyz,kpxe";
    }
}

subnet 10.10.0.0 netmask 255.255.240.0 {
    option broadcast-address    10.10.3.255;
    option subnet-mask          255.255.240.0;
    option routers              10.10.0.1;
    pool {
        failover peer "dhcp-failover";
        range 10.10.1.20 10.10.2.200;
    }
    if exists user-class and ( option user-class = "iPXE" ) {
        filename "http://boot.netboot.xyz/menu.ipxe";
    } elsif option arch = encode-int ( 16, 16 ) {
        filename "http://boot.netboot.xyz/ipxe/netboot.xyz.efi";
        option vendor-class-identifier "HTTPClient";
    } elsif option arch = 00:07 {
        filename "netboot.xyz.efi";
    } else {
        filename "netboot.xyz,kpxe";
    }
}

subnet 10.20.0.0 netmask 255.255.240.0 {
    option broadcast-address    10.20.3.255;
    option subnet-mask          255.255.240.0;
    option routers              10.20.0.1;
    pool {
        failover peer "dhcp-failover";
        range 10.20.0.20 10.20.2.200;
    }
    if exists user-class and ( option user-class = "iPXE" ) {
        filename "http://boot.netboot.xyz/menu.ipxe";
    } elsif option arch = encode-int ( 16, 16 ) {
        filename "http://boot.netboot.xyz/ipxe/netboot.xyz.efi";
        option vendor-class-identifier "HTTPClient";
    } elsif option arch = 00:07 {
        filename "netboot.xyz.efi";
    } else {
        filename "netboot.xyz,kpxe";
    }
}

subnet 10.30.0.0 netmask 255.255.255.0 {
    option broadcast-address    10.30.0.255;
    option subnet-mask          255.255.255.0;
    option routers              10.30.0.1;
    pool {
        failover peer "dhcp-failover";
        range 10.30.0.20 10.30.0.200;
    }
}

subnet 10.100.0.0 netmask 255.255.240.0 {
    option broadcast-address    10.100.3.255;
    option subnet-mask          255.255.240.0;
    option routers              10.100.0.1;
    pool {
        failover peer "dhcp-failover";
        range 10.100.1.100 10.100.2.200;
    }
    if exists user-class and ( option user-class = "iPXE" ) {
        filename "http://boot.netboot.xyz/menu.ipxe";
    } elsif option arch = encode-int ( 16, 16 ) {
        filename "http://boot.netboot.xyz/ipxe/netboot.xyz.efi";
        option vendor-class-identifier "HTTPClient";
    } elsif option arch = 00:07 {
        filename "netboot.xyz.efi";
    } else {
        filename "netboot.xyz,kpxe";
    }
}
```

## Encrypted configuration file

This role uses global variables from `./inventory/group_vars/all.sops.yaml` and the very first time we create it, it must be encrypted manually once with SOPS:

```bash
sops -e -i ./inventory/group_vars/all.sops.yaml
```

After that the VS Code extension `signageos/vscode-sops` automattically decrypts and reencrypts the file whenever it is edited (and Ansible has a plugin to decrypt variables when loaded - specified in `ansible.cfg`).
For this to work we need to have `sops` and `age` installed and a key generated and stored in `~/.sops/key,txt`.
In addition, a SOPS configuration file is created in the `./inventory` directory which contains the `age` public key:

```yaml
  # file: ./inventory/.sops.yaml
  creation_rules:
    - path_regex: .*vars/.*
      age: "age109fzapgarv59gdxv6zqmwgn0j5kxmfz8dhrz9lrqvky046jxafmse38kvj"
```

## Inventory example

```yaml
# file: inventory/group_vars/all.sops.yaml
dns_master:
  hosts:
    gandalf:
      ansible_port: 22
      ansible_host: 10.0.0.10
dns_slave:
  hosts:
    sauron:
      ansible_port: 22
      ansible_host: 10.0.0.8
dns_server:
  children:
    dns_master:
    dns_slave:
```

### Roles directory

To ensure the local roles that are specific for the playbook are dfistinguishable, place them in the directory `roles/local` to ensure onlyt hese are commited to the repository.
Roles like this that are downloaded via `requirements.yaml` are placed directly in the `roles` directory and not included in the playbook repository.
A simple `roles/.gitignore` will take care of that:

```ini
#Ignore everything in roles dir...
/*
# ... but current file...
!.gitignore
# ... external role requirement file
!requirements.yml
# ... and configured custom/local roles
!local*/
```

### Ansible configuration

To use the automated encryption/decryption, there are a few additional requirements for the Ansible configuration to be specified in the `ansible.cfg` file.

```ini
[defaults]
...
vars_plugins_enabled = host_group_vars,community.sops.sops
roles_path = roles
[community.sops]
vars_stage = inventory
[inventory]
enable_plugins = host_list, script, auto, yaml, ini
```

## Usage of this role

To use this role, include the following section in a `requirements.yml` file in the local `roles` directory:

```yaml
# Include the 'bind` role from GitHub
- src: git@github.com:crazyelectron-io/ansible-role-dhcp.git
  scm: git
  version: main
  name: dhcp
```

> Only include the 'top' roles, dependencies - when listed in `meta/main.yml` of the imported role - will be downloaded automatically.

To retrieve roles like this in your project, run `ansible-galaxy install -r requirements.yaml`.
Because these roles will not be updated locally when the repository is changed, to refresh an already retrieved role use `ansible-galaxy install -f -r requirements.yaml`

## Dependencies

BIND9 role `bind`.

## Workflow to deploy from a playbook

1. Add the dhcp dependancy to `requirements.yaml`.
2. Download the external roles: `ansible-galaxy install -r roles/requirements`
3. Add a `.gitignore` in the `roles` directory to ensure this downloaded roles are not committed to the playbook repository (see `.gitignore-example`).
4. Add the role variables to `inventorygroup_vars\all.sops.yaml` (see `example.all.sops.yaml`).
5. Setup the inventory hosts file `hosts.yaml` (see `example.hosts.yaml`).
6. Launch your playbook: `ansible-playbook -i inventory some_playbook.yml [-u ANSIBLE_USER]`

DHCP reserved IP addresses file template:

```yaml
XXX
```



