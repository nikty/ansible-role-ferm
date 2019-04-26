Simplistic ansible role to manage Ferm
==========

Manage and configure the ferm firewall.

With this role you can manage and create a set of global firewall rules and merge them with:
- Firewall rules for a specific group
- Firewall rules for a specific host


Requirements
------------

hash\_behaviour set to merge in ansible.cfg

```ini
[defaults]
hash_behaviour = merge
```


Configuration files and variables structure
-------------------------------------------


Example configuration:

`group_vars/all.yml` or `roles/ferm/vars/main.yml` contains your global firewall rules that will apply to all your hosts
  
`group_vars/webservers.yml` Firewall rules in this file will only apply to members of the `webservers` group.
  
`host_vars/webserver01.yml` can contain rules for a specific host.


Role Variables
--------------
Each key under `ferm_rules` will be created as a configuration file for ferm under `/etc/ferm/ferm.d`. Files in that directory will be handled in alphabetical order. To keep things simple, I maintain the following naming scheme:

- 01-policies
- 02-icmp
- 10-mgmt
- 30-webserver01
- 50-webservers
- 99-reject-all

Duplicate names will be overridden in the order of variable importancy.

Examples
========

Below is a set of firewall rules that will apply to all your servers, together with some firewall rules that will only apply to members of a group called `webservers` and some firewall rules that only apply to a specific host.

Example inventory:
```
[all]
server1
server2
server3
docker01
docker02
webserver01
webserver02

[webservers]
webserver01
webserver02
```

Global firewall rules
---------------------

In `group_vars/all.yml` or `roles/ferm/vars/main.yml`:
```yaml
---
ferm_rules:
  01-policies:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: policy ACCEPT
        - rule: mod state state INVALID DROP
          comment: "Drop invalid packets"
        - rule: mod state state (ESTABLISHED RELATED) ACCEPT
          comment: "Allow already established sessions"
        - rule: interface lo ACCEPT
          comment: "Allow traffic from localhost interface"

    - chain: OUTPUT
      domains: [ip, ip6]
      rules:
        - rule: policy ACCEPT

    - chain: FORWARD
      domains: [ip, ip6]
      rules:
        - rule: policy DROP

  02-icmp:
    - chain: INPUT
      domains: [ip]
      rules:
        - rule: proto icmp icmp-type (3 8 11) ACCEPT
          comment: "Allow dest unreachable, ping and time exceeded messages"
    - chain: INPUT
      domains: [ip]
      rules:
        - rule: proto ipv6-icmp ACCEPT
          comment: "Allow all ipv6 icmp messages"

  10-mgmt:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: proto tcp dport ssh saddr (192.168.178.0/28 2001:db8::/64) ACCEPT
          comment: "Allow SSH from my trusted IPs"
          
  99-reject:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: REJECT reject-with icmp-host-prohibited
```

Group firewall rules
--------------------

The following rules are applied to all members of the `webservers` group. They are placed in `group_vars/webservers.yml`:
```yaml
---

ferm_rules:
  50-webservers:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: proto tcp dport (http https) ACCEPT
          comment: "Allow HTTP and HTTPS"
        - rule: proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT
          comment: "Allow developers to our admin panels"
```

Our webservers will have these ports opened on top of the global firewall rules.


Host specific firewall rules
----------------------------

Webserver01 needs a temporary firewall rule, we can place this in `host_vars/webserver01.yml`:
```yaml
---

ferm_rules:
  30-webserver01:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: proto tcp dport (ssh) saddr 198.51.100.128/25 2001:db8::beef:/64) ACCEPT
          comment: "Temporary senior developers SSH access"
```


Result
------

`webserver02` gets an additional rule file on top of the global default called:

/etc/ferm/ferm.d/50-webservers
```conf
domain (ip ip6) table filter {
  chain INPUT {
     # Allow HTTP/HTTPS
     mod comment comment "Allow HTTP and HTTPS" proto tcp dport (http https) ACCEPT;

     # Allow developers to dev sites
     mod comment comment "Allow developers to dev sites" proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT;
    }
}
```

`webserver01` gets two files on top of the global default:

/etc/ferm/ferm.d/50-webservers
```conf
domain (ip ip6) table filter {
  chain INPUT {
     # Allow HTTP/HTTPS
     mod comment comment "Allow HTTP/HTTPS" proto tcp dport (http https) ACCEPT;

     # Allow developers to dev sites
     mod comment comment "Allow developers to our admin panels" proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT;
    }
}
```

/etc/ferm/ferm.d/30-webserver01
```conf
domain (ip ip6) table filter {
  chain INPUT {
     # Allow senior developers SSH access
     mod comment comment "Temporary senior developers SSH access" proto tcp dport (ssh) saddr (198.51.100.128/25 2001:db8::beef:/64) ACCEPT;
    }
}
```
