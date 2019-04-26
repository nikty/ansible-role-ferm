Ferm-Firewall
==========

Manage and configure the ferm firewall. Send separate configuration files per groups or hosts
You may need to change ansible `hash` from replace to merge .

Requirements
------------

hash\_behaviour set to merge in ansible.cfg

```conf
[defaults]
hash_behaviour = merge
```


Configuration files and variables structure
-------------------------------------------

roles/ferm/vars/main.yml contains a set of basic sane firewall rules with all ports closed


Example configuration:

 - group\_vars/all.yml
  - this can have a ferm\_rules defined - used on all hosts
 - group\_vars/webservers.yml
  - Place ferm rules that you want to apply to servers in a group `webservers`.
 - host\_vars/webserver01.yml
  - Here go ferm rules for one specific host called `webserver01`.

Role Variables
--------------
To configure ferm, you need to provide a key to associate a set of rules to a role/software. This way, rules splited in multiple var-files won't overwrite each other.
By default, if domains isn't defined, it will apply rules to ip6 and ip domains.
Configuration example:

```yaml
---
# Your default ferm rules for all hosts
ferm_rules:
# Creates a file called /etc/ferm/ferm.d/050-example
  050-example:
    - chain: INPUT
      rules:
        - {rule: "policy DROP;",  comment: "global policy"}
        - {rule: "mod state state INVALID DROP;", comment: "connection tracking: drop"}
        - {rule: "mod state state (ESTABLISHED RELATED) ACCEPT;", comment: "connection tracking"}
        - {rule: "interface lo ACCEPT;", comment: "allow local packet"}
        - {rule: "proto icmp ACCEPT;", comment: "respond to ping"}
        - {rule: "proto tcp dport ssh ACCEPT;", comment: "allow SSH connections"}
    # Different set of rules on ip / ip6
    - chain: OUTPUT
      domains:
        - ip
      rules:
        - rule: "policy ACCEPT;"
          comment: global policy
    - chain: OUTPUT
      domains:
        - ip6
      rules:
        - rule: "policy DROP;"
          comment: global policy ip6

    - chain: FORWARD
      domains: [ip, ip6]
      rules:
        - rule: "policy DROP;"
          comment: global policy
        - rule: "mod state state INVALID DROP;"
          comment: "connection tracking: drop"
        - rule: "mod state state (ESTABLISHED RELATED) ACCEPT;"
          comment: "connection tracking"

```

Dependencies
------------
 - None

Example Playbook
----------------
Ferm rules are hash instead of array. The main reason is to be able to merge hashes when configure same host with different roles.

Inventory:
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

[docker-engines]
docker01
docker02


```

group\_vars/webservers.yml:
```yaml
---

ferm_rules:
  50-webservers:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: proto tcp dport (http https) ACCEPT
          comment: "Allow HTTP/HTTPS"
        - rule: proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT
          comment: "Allow developers to dev sites"

```


host\_vars/webserver01.yml:
```yaml
---

ferm_rules:
  50-webservers-mgmt:
    - chain: INPUT
      domains: [ip, ip6]
      rules:
        - rule: proto tcp dport (ssh) saddr 198.51.100.128/25) ACCEPT
          comment: "Allow senior developers SSH access"

```

Result:


`webserver02` gets a file called:

/etc/ferm/ferm.d/50-webservers
```conf
domain (ip ip6) table filter {
  chain INPUT {
     # Allow HTTP/HTTPS
     mod comment comment "Allow HTTP/HTTPS" proto tcp dport (http https) ACCEPT;

     # Allow developers to dev sites
     mod comment comment "Allow developers to dev sites" proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT;
    }
}
```

`webserver01` gets two files:

/etc/ferm/ferm.d/50-webservers
```conf
domain (ip ip6) table filter {
  chain INPUT {
     # Allow HTTP/HTTPS
     mod comment comment "Allow HTTP/HTTPS" proto tcp dport (http https) ACCEPT;

     # Allow developers to dev sites
     mod comment comment "Allow developers to dev sites" proto tcp dport (8080 8443) saddr (192.168.178.0/24 198.51.100.0/24 2001:db8::/48) ACCEPT;
    }
}
```

/etc/ferm/ferm.d/50-webservers-mgmt
```conf
domain (ip ) table filter {
  chain INPUT {
     # Allow senior developers SSH access
     mod comment comment "Allow senior developers SSH access" proto tcp dport (ssh) saddr 198.51.100.128/25) ACCEPT;
    }
}
```


