Starting from v1.6.0-rc.1 you can list and configure system firewall rules from the GUI.

The old configuration format will be supported for some time, but you're encouraged to migrate your rules to the new format.

The supported firewall is nftables. If you're using iptables, you won't be able to configure rules from the GUI. You can still add them to the configuration file.

What's new
---

- Allow to configure chains' policies (>= v1.6.0rc1)

You can apply a restrictive firewall policy for inbound connections by setting the inbound policy to Deny. This policy will add two extra rules to allow outbound connections (allow established connections + allow connections to localhost, needed for many services like DNS resolvers).

![image](https://user-images.githubusercontent.com/2742953/166963755-58eb268a-0d24-44f4-89a8-a25a252d1207.png)


- Add basic firewall rules (>= v1.6.0rc1)

Besides restricting what applications can access the internet, now you can configure general firewall rules.

![image](https://user-images.githubusercontent.com/2742953/166965334-939247ba-2002-4d7b-9232-ea2b225f02a8.png)

For now you can only configure basic inbound or outbound connections (TCP and UDP). For example you can allow inbound SSH and outbound WireGuard

![image](https://user-images.githubusercontent.com/2742953/166966001-2ee27591-5271-48a2-a474-91bf8758c4fb.png)


Firewall configuration format
---

The firewall configuration offers much more options to configure the system firewall.


Chains
---

Supported Chains are the ones defined according to this matrix:

       Table 6. Standard priority names, family and hook compatibility matrix
       ┌─────────┬───────┬────────────────┬─────────────┐
       │Name     │ Value │ Families       │ Hooks       │
       ├─────────┼───────┼────────────────┼─────────────┤
       │raw      │ -300  │ ip, ip6, inet  │ all         │
       ├─────────┼───────┼────────────────┼─────────────┤
       │mangle   │ -150  │ ip, ip6, inet  │ all         │
       ├─────────┼───────┼────────────────┼─────────────┤
       │dstnat   │ -100  │ ip, ip6, inet  │ prerouting  │
       ├─────────┼───────┼────────────────┼─────────────┤
       │filter   │ 0     │ ip, ip6, inet, │ all         │
       │         │       │ arp, netdev    │             │
       ├─────────┼───────┼────────────────┼─────────────┤
       │security │ 50    │ ip, ip6, inet  │ all         │
       ├─────────┼───────┼────────────────┼─────────────┤
       │srcnat   │ 100   │ ip, ip6, inet  │ postrouting │
       └─────────┴───────┴────────────────┴─────────────┘

https://www.netfilter.org/projects/nftables/manpage.html#lbAQ


The format is as follow:

```json
        {
          "Name": "input",
          "Table": "filter",
          "Family": "inet",
          "Priority": "",
          "Type": "filter",
          "Hook": "input",
          "Policy": "accept",
          "Rules": [
          ]
```

`Name` is just the name of the chain. It can be whatever you want.

Possible options that you can combine to create new chains:

| Field | Options |
|-------|---------|
|Family| ip, ip6, inet, netdev, bridge|
|Priority| not used|   
|Type| filter, mangle, conntrack, natdest, natsource, raw, security, selinux|
|Hook| prerouting, input, output, postrouting, forward, ingress|
|Policy| drop, accept|

All the possible options are described here:
https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks#Priority_within_hook
    
Rules
---
    
Example:
```json
    {
              "UUID": "97dd2578-2f29-4bcc-a296-f4358c16501d",
              "Enabled": true,
              "Position": "0",
              "Description": "Allow UDP navigation when INPUT policy is DROP",
              "Expressions": []
              "Target": "accept",
              "TargetParameters": ""
    }
```

 |Fields | Description|
 |-------|------------|
 |UUID|Unique identifier for this rule. If it's empty, a new temporal UUID will be generated|
 |Enabled| true or false |
 |Position| no used yet |
 |Description| Description of the rule|
 |Expressions| List of options to match against connections: tcp dport 22 (see below)|
 |Target| Action applied on the connection: accept, deny, reject, return, jump, goto, stop, tproxy, redirect, dnat, snat|
 |TargetParameters|Parameters of the given Target. For example: Target -> redirect, TargetParameters -> to :8080|
    
 Rules expressions
 ---
 
 Expressions are a list of statements that represent the actions to be performed on the connections. They can alter control flow (return, jump to a different chain, accept or drop the packet) or can perform actions, such as logging, rejecting a packet, etc. 
 
 https://www.netfilter.org/projects/nftables/manpage.html#lbCV
 https://wiki.nftables.org/wiki-nftables/index.php/Building_rules_through_expressions
 
 Example:
 ```json
               "Expressions": [
                {
                  "Statement": {
                    "Op": "",
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "state",
                        "Value": "invalid"
                      }
                    ]
                  }
                },
                {
                  "Statement": {
                    "Op": "",
                    "Name": "log",
                    "Values": [
                      {
                        "Key": "prefix",
                        "Value": "invalid-in-packet"
                      }
                    ]
                  }
                }
              ],
```
 
 Each statement has different values (Key and Value field). Not all official statements are supported, only the ones described on the following table:
 
 |Statement Name|Values|Description|Example|
 |---------|------|-----------|-------|
 |log| Key: prefix . TODO: flags, log level|Logs connections to the system with the given prefix|Name: log, Key: prefix, Value: "ssh out"|
 |iifname, oifname|Key: eth0, wlp3s0, etc.. (network interface name), Value field is ignored in this case.|Matches the input network interface (iifname) or the output one (oifname)|Name: iifname, Key: lo|
 |ip,ip6|Key: daddr, saddr|Matches dest or source address. You can specify an IP, a range of IPs or IPs separated by commas|Name: ip, Key: daddr, Value: 127.0.0.1|
 |limit| Key: units, rate-units, time-units | rate-limit connections. For example: limit HTTPS downloads to 1MB/s| tcp sport 443 limit rate over 1 mbytes/second drop |
 |udp,tcp,sctp,dccp|Key: sport,dport| Matches against dest or source port on the given network protocol. You can specify ports separated by commas and port ranges.| Name: tcp, Key: dport, Value: 22|
 |quota|Key: quota|Applies the given verdict on connections matching certain criteria: like when going over a given mbytes, gbytes, etc|Name: quota, Key: over, Key: "mbytes", Value: "100"|
 |counter| Key: name||Name: counter, Key: name, Value: "dport 22 counter"|
 |ct|Key: state, mark; Value: invalid, new, established, related|Matches connections on the conntrack table||
 |meta|Key: mark|||
 
 The field `Op` is the operator to use on the statement: ==, >=, <=, >, <, != . If it's empty, by default the equal operator (==) will be used.
 
 Examples of supported statements
 ---
 
 log:
 ```json
                   "Statement": {
                    "Op": "",
                    "Name": "log",
                    "Values": [
                      {
                        "Key": "prefix",
                        "Value": "invalid-in-packet"
                      }
                    ]
                  }
 ```
 
 ---
 
 iifname, oifname:
 ```json
                  "Statement": {
                    "Op": "",
                    "Name": "iifname",
                    "Values": [
                      {
                        "Key": "lo",
                        "Value": ""
                      }
                    ]
                  }
```

---

ip + daddr, IP ranges (network ranges not supported)
```json
                  "Statement": {
                    "Op": "!=",
                    "Name": "ip",
                    "Values": [
                      {
                        "Key": "daddr",
                        "Value": "192.168.2.100-192.168.2.200"
                      }
                    ]
                  }
```

ip + saddr + multiple IPs separated by commas
```json
                  "Statement": {
                    "Op": "",
                    "Name": "ip",
                    "Values": [
                      {
                        "Key": "saddr",
                        "Value": "192.168.2.1,192.168.2.2"
                      }
                    ]
                  }
```

ip + daddr + single IP
```json
                  "Statement": {
                    "Op": "",
                    "Name": "ip",
                    "Values": [
                      {
                        "Key": "daddr",
                        "Value": "192.168.2.100"
                      }
                    ]
                  }
```

---

tcp + dport, single dport
```json
                  "Statement": {
                    "Op": "",
                    "Name": "tcp",
                    "Values": [
                      {
                        "Key": "dport",
                        "Value": "443"
                      }
                    ]
                  }
```

udp + dport + operator, dports range
```json
                  "Statement": {
                    "Op": ">=",
                    "Name": "udp",
                    "Values": [
                      {
                        "Key": "dport",
                        "Value": "15000-20000"
                      }
                    ]
                  }
```

tcp + dport, multiple ports separated by commas
```json
                  "Statement": {
                    "Op": "",
                    "Name": "tcp",
                    "Values": [
                      {
                        "Key": "dport",
                        "Value": "8080,8081"
                      }
                    ]
                  }
```

---

Rate-limit HTTPS downloads to 1MB/s (table filter, chain input)

https://wiki.nftables.org/wiki-nftables/index.php/Rate_limiting_matchings 

https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes#Limit

```json
                {
                  "Statement": {
                    "Op": "==",
                    "Name": "tcp",
                    "Values": [
                      {

                        "Key": "sport",
                        "Value": "443"
                      }
                    ]
                  }
                },
                {
                  "Statement": {
                    "Op": "==",
                    "Name": "limit",
                    "Values": [
                      {
                        "Key": "over",
                        "Value": ""
                      },
                      {
                        "Key": "units",
                        "Value": "1"
                      },
                      {
                        "Key": "rate-units",
                        "Value": "mbytes"
                      },
                      {
                        "Key": "time-units",
                        "Value": "second"
                      }
                    ]
                  }
                },
```

---

Apply a quota on a connection when the given connection exceeds 1GB. When it exceeds the defined limit, the verdict you specify will be applied (deny, accept, etc) https://wiki.nftables.org/wiki-nftables/index.php/Quotas
```json
                  "Statement": {
                    "Op": "",
                    "Name": "quota",
                    "Values": [
                      {
                        "Key": "over",
                        "Value": ""
                      },
                      {
                        "Key": "gbytes",
                        "Value": "1"
                      }
                    ]
                  }
```

---

count packets of a given connection:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "counter",
                    "Values": [
                      {
                        "Key": "packets",
                        "Value": ""
                      },
                      {
                        "Key": "name",
                        "Value": "dport 443 counter"
                      }
                    ]
                  }
```

---

matching conntrack states:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "state",
                        "Value": "invalid"
                      }
                    ]
                  }
```

matching multiple conntrack states:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "state",
                        "Value": "related"
                      },
                      {
                        "Key": "state",
                        "Value": "established"
                      }
                    ]
                  }
```

matching multiple conntrack states II:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "state",
                        "Value": "related,established"
                      },
                    ]
                  }
```

matching a conntrack mark (decimal value):
```json
                 "Statement": {
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "mark",
                        "Value": "666"
                      }
                    ]
                  }
```

setting a conntrack mark on packets (decimal value):
```json
                  "Statement": {
                    "Name": "ct",
                    "Values": [
                      {
                        "Key": "set",
                        "Value": ""
                      },
                      {
                        "Key": "mark",
                        "Value": "666"
                      }
                    ]
                  }
```

---

Apply limits on connections, 1MB/s (you can combine it with tcp -> sport 443 to limit download bandwidth for example):
```json
                  "Statement": {
                    "Op": "",
                    "Name": "limit",
                    "Values": [
                      {
                        "Key": "units",
                        "Value": "1"
                      },
                      {
                        "Key": "rate-units",
                        "Value": "mbytes"
                      },
                      {
                        "Key": "time-units",
                        "Value": "second"
                      }
                    ]
                  }
```

// TODO: add burst example

---

meta, set priority:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "meta",
                    "Values": [
                      {
                        "Key": "priority",
                        "Value": "1"
                      }
                    ]
                  }
```

meta, match packets by mark:
```json
                 "Statement": {
                    "Op": "",
                    "Name": "meta",
                    "Values": [
                      {
                        "Key": "mark",
                        "Value": "122"
                      }
                    ]
                  }
```

meta, set mark on packets:
```json
                  "Statement": {
                    "Op": "",
                    "Name": "meta",
                    "Values": [
                      {
                        "Key": "set",
                        "Value": ""
                      },
                      {
                        "Key": "mark",
                        "Value": "57005"
                      }
                    ]
                  }
```