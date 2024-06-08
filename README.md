# DHCP Role

## Create subnets and reserved IPs

Ansible task:

```yaml
# file: ./roles/local/dhcp-config/tasks/main.yaml
# synopsis: deploy isc-dhcp-server in a high-availability setup
---
- set_fact:
    update_subnets: false
- block:
    - name: DHCP | Get stats of the target subnet config file
      ansible.builtin.stat:
        path: "/etc/dhcp/dhcp-subnets.conf"
      register: config_subnets_stats
    # - debug:
    #     var: config_subnets_stats
    - name: DHCP |Get stats of the template subnet file
      ansible.builtin.stat:
        path: "{{ role_path }}/templates/dhcp-subnets.conf.j2"
      register: template_subnets_stats
      become: false
      delegate_to: localhost
    - name: DHCP | Check if the subnet config file is newer than the template
      set_fact:
        update_subnets: true
      when: not config_subnets_stats.stat.exists or config_subnets_stats.stat.mtime < template_subnets_stats.stat.mtime
  when: inventory_hostname in groups.dns_servers

- name: DHCP | Define DHCP managed subnets
  ansible.builtin.template:
    src: "dhcp-subnets.conf.j2"
    dest: "/etc/dhcp/dhcp-subnets.conf"
    owner: "root"
    group: "root"
    mode: "0644"
  notify:
    - Restart isc-dhcp-server

- set_fact:
    update_reserved: false

- block:
    - name: DHCP | Get stats of the target reserved addresses file
      ansible.builtin.stat:
        path: "/etc/dhcp/dhcp-reserved.conf"
      register: config_reserved_stats
    - name: DHCP | Get stats of the template reserved addresses file
      ansible.builtin.stat:
        path: "{{ role_path }}/templates/dhcp-reserved.conf.j2"
      register: template_reserved_stats
      become: false
      delegate_to: localhost
    - name: DHCP | Check if the reserved addressed config file is newer than the template
      set_fact:
        update_reserved: true
      when: not config_reserved_stats.stat.exists or config_reserved_stats.stat.mtime < template_reserved_stats.stat.mtime
  when: inventory_hostname in groups.dns_servers

- name: DHCP | Define reserved IP addresses
  ansible.builtin.template:
    src: "dhcp-reserved.conf.j2"
    dest: "/etc/dhcp/dhcp-reserved.conf"
    owner: "root"
    group: "root"
    mode: "0644"
  notify:
    - Restart isc-dhcp-server

- block:
  - name: "DHCP | Get services info"
    ansible.builtin.service_facts:
  - name: DHCP | Fail if isc-dhcp-server is not running
    ansible.builtin.fail:
      msg: "DHCP configuration failed, service not running!"
    when: services['isc-dhcp-server'].state != 'running'
  when: inventory_hostname in groups.dns_servers
```

Ansible template for subnet definitions:

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

Ansible template for reserved IP configuration:

```yaml
# file: /etc/dhcpd/dhcp-reserved.conf.j2
# synopsis: defines the reserved IP addresses for all supported VLANs
# {{ ansible_managed }}

host host1 {
  hardware ethernet 11:22:33:44:55:66;
  fixed-address 10.20.2.212;
}
host host2 {
  hardware ethernet 11:22:33:44:55:77;
  fixed-address 10.20.2.213;
}

```
