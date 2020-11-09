---
# Ansible Role: openbsd-router

This role configures the OpenBSD router.

The packages will be installed: arping, htop, pftop, mtr, rsync, vim, wget.
This role will also install the latest OpenBSD binary patches.

### Default variables

```
bgpd: false
dhcpd: false
dns_servers: ['8.8.8.8', '8.8.4.4']
iked: false
mirror_url: https://ftp.yandex.ru/openbsd
ntpd: true
ntpd_conf_tpl: "ntpd.conf.j2"
ntpd_servers: pool.ntp.org
ospfd: false
pf: false
services_default:
  - cron
  - pflogd
  - smtpd
  - sshd
  - syslogd
  - unbound
snmpd: false
snmpd_listen_on: 127.0.0.1
snmpd_community: public
snmpd_conf_tpl: "snmpd.conf.j2"
sysctl_default:
  net.inet.ip.forwarding: 1
  net.inet.carp.preempt: 1
timezone: UTC # see /usr/share/zoneinfo
```

### Mandatory variables

* interfaces (dictionary) - the key is the name of the interface, the value is the interface settings, 
see hostname.if(5)
* dhcpd_options (list) - list of dhcpd interfaces, required if dhcpd is true

### Optional variables

* bgpd (bool) - enables bgpd. This option requires bgpd.conf.j2 template, see bgpd.conf(5)
* dhcpd (bool) - enables dhcpd. This option requires the definition of dhcpd_options 
and dhcpd.conf.j2 template, see dhcpd.conf(5)
* dns_servers (list) - ip addresses of nameservers. Default settings: `['8.8.8.8', '8.8.4.4']`
* hostname (scalar) - the symbolic name of the host machine, see myname(5)
* iked (bool) - enables iked. This option requires the defenition iked variables (see example below)
* ntpd (bool) - enables ntpd. This option requires the configuration within ntpd_conf variable or 
  in the template file defined in ntpd_conf_tpl variable.
* ntpd_conf (scalar) - ntpd configuration (see also template if it's not defined).
* ntpd_conf_tpl (scalar) - ntpd template filename
* ntpd_listen_on (list) - addreses which ntpd must be listen on. By default it's not listen on interfaces.
* ntpd_servers (scalar) - ntpd server to synchronize to. (works for temaplate only).
* mygate (scalar) - the address of the gateway host, see mygate(5). The gateway will be removed
if bgpd defined.
* ospfd (bool) - enables ospfd. This option requires ospfd.conf.j2 template, see ospfd.conf(5)
* pf (bool) - enables packet filter. This option requires pf.conf.j2 template, see pf.conf(5)
* services (list) - services that you need to enable. Overrides default ones. See default values below.
* snmpd (bool) - enables snmpd.
* snmpd_listen_on - address which snmpd must be listen on. Default is 127.0.0.1.
* snmpd_community - community name. Default is public.
* snmpd_conf_tpl - snmpd template filename
* sysctl (dictionary) - sysctl key/value pairs. See default values below.
* timezone (scalar) - timezone, see /usr/share/zoneinfo

### Example of usage
This is an exmaple of a `router1` with the private address 192.168.0.5 (router1 behind router with public
address 1.1.1.1) that connects to `router2` with public ip 2.2.2.2. The internal network of a `router1`
is 192.168.0.0/24 and internal network of `router2` is 192.168.1.0/24 and 192.168.10.0/24. You need to copy
the contents of the file `/etc/iked/local.pub` at router2 and paste it into this playbook for router1: 

host_vars\router1.yml:
```
hostname: ipsec.local
interfaces:
  vmx0: |
    inet 192.168.0.5 255.255.255.0 NONE description "VPN"
mygate: 192.168.0.1
dns_servers: ['127.0.0.1', '8.8.8.8', '8.8.4.4']
pf: yes
ntpd: yes
ntpd_listen_on: "{{ hostvars[inventory_hostname].ansible_interfaces | select('match','carp.*') | difference(['carp2', 'carp3', 'carp9']) | map('regex_replace', '^(.*)$', 'ansible_\\1') | map('extract', hostvars[inventory_hostname], ['ipv4', 0, 'address']) | list }}"
iked: yes
iked_peers:
  2.2.2.2:
    label: remote-office
    mode: active
    srcid: 1.1.1.1
    dstid: 2.2.2.2
    flows:
      - from 192.168.0.0/24 to 192.168.1.0/24
      - from 192.168.0.0/24 to 192.168.10.0/24
    public_key: |
        The remote RSA public key of 2.2.2.2
        -----BEGIN PUBLIC KEY-----
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2bpKfHK7t30a6GG7dUwd
        TXyQ7CGwNRRPZGmw3xCRHUr8mbXklB3aL4Sn26xbi5usnP6nLHiE03hLBOoIfRrT
        xHqVBczs7WH95lPFxQjrcotduwXxFhK7KBLrcsW0e++zZRvEScmYbeqTiLKDhDMG
        h7FJvw8hwgbeXW5y2s5EzW7RcQkaJm/AbOnm2bVSLZZgmJwHuA4pubjUOXfZPu1l
        hF9ZeJDBG72VXhmHU77NTD6+SBk2k1FAfhrEs7mlJ37MAMfekRo0Hwa33hSF+a2x
        Rk2L9/EswhYPXQuvNoxbEDtjuQj7XmxrZVqvNqJX4AxYE1pBFZ5v4nWw3C4PQBkf
        1wIDAQAB
        -----END PUBLIC KEY-----
```

File templates\router1\pf.conf.j2:
```jinja2
# Packet filter on {{ inventory_hostname }}

table <firewall> const { self }

set block-policy drop
set skip on { lo enc0 }

pass out quick from <firewall>

block log
pass in quick proto udp from 2.2.2.2 to <firewall> port { isakmp, ipsec-nat-t }
pass in quick proto esp from 2.2.2.2 to <firewall>
pass in quick inet proto tcp to <firewall> port ssh
pass quick from 192.168.0.0/24 to { 192.168.1.0/24 192.168.10.0/24 }
pass quick from { 192.168.1.0/24 192.168.10.0/24 } to 192.168.0.0/24
```

### License

MIT

### Author Information

dynax60