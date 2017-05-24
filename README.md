# OVN on air

Let's make [OVN, Open Virtual Network](http://openvswitch.org/support/dist-docs/ovn-architecture.7.html),
on [MacBook Air](https://github.com/keinohguchi/arch-on-air/blob/master/README.md)!

1. [Setup](#setup)
2. [Playbooks](#playbooks)

## Setup

I'm using KVM/libvirt to have those three KVM instances on Air and
reachable through `hv11`, `hv12`, and `hv13`, respectively.  And
to make the playbook clean, I'm using ubuntu 16.04 for all three
KVM instances.

```
                      +------------+
                      |            |
                      |    hv11    |
                      |            |
                      +------+-----+
                             |
                             |
                             |
         +------------+      |     +------------+
         |            |      |     |            |
         |    hv12    |      |     |    hv13    |
         |            |      |     |            |
         +------+-----+      |     +------+-----+
                |            |            |
                |            |            |
+---------------+------------+------------+----------------+
|                                                          |
|                          air                             |
|                                                          |
+----------------------------------------------------------+
```

## Playbooks

1. [build.yml](#buildyml)
2. [install.yml](#installyml)
3. [setup.yml](#setupyml)
4. [configure.yml](configureyml)

### build.yml

[build.yml](build.yml) is to setup the build environment on hv11, as explained in
[Documentation/intro/install/debian.rst](https://github.com/openvswitch/ovs/blob/master/Documentation/intro/install/debian.rst).

```
air$ ansible-playbook build.yml
```

This playbook will:

1. setup the `hv11` as a build machine
2. git clone the latest OvS from the github
3. build OvS/OVN
4. download the deb package from the build machine to the local `/tmp` directory

### install.yml

[install.yml](install.yml) will install the required deb packages to all
the node.

```
air$ ansible-playbook install.yml
```

This playbook will upload the deb packages and install on all the nodes.

### setup.yml

[setup.yml](setup.yml) will setup OvS/OVN for virtual networking.

```
air$ ansible-playbook setup.yml
```

This playbook will:

1. Open up OVN ports on the `central` nodes
2. Configure integration bridge, `br-int` on `chassis` nodes
3. Configure OVN central node IP on `chassis` nodes
4. Configure OVN encapsulation type/IP on `chassis` nodes

After the playbook run, you should be able to see the OVN connection, `TCP/6642`
between the central node and the chassis nodes, as shown below.

```
air$ ssh hv11 netstat -antp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:6641            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:6642            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0     72 192.168.122.111:22      192.168.122.1:60624     ESTABLISHED -
tcp        0      0 192.168.122.111:22      192.168.122.1:60618     ESTABLISHED -
tcp        0      0 192.168.122.111:6642    192.168.122.112:47680   ESTABLISHED -
tcp        0      0 192.168.122.111:6642    192.168.122.113:50234   ESTABLISHED -
tcp6       0      0 :::22                   :::*                    LISTEN      -
air$
```

### configure.yml

Now, all the nodes got configured and talking each other.  Let's configure the OVN!

```
air$ ansible-playbook configure.yml
```

This playbook will:

1. Create logical switches for each tenant, e.g. red, blue, green
2. Create logical ports, e.g. red1, red2, under the appropriate switches
3. Setup MAC address and MAC address based port security on the appropriate switches

Here is the result of the logical switches after the playbook run.

```
air$  ssh hv11 sudo ovn-nbctl show
switch 23865fe0-92a2-4289-867b-36237eeba077 (green)
    port green2
        addresses: ["00:00:00:13:0c:01"]
    port green1
        addresses: ["00:00:00:12:0c:01"]
switch 809d63e6-32d6-4698-b9d2-b973f951c69c (red)
    port red2
        addresses: ["00:00:00:13:0a:01"]
    port red1
        addresses: ["00:00:00:12:0a:01"]
switch 20af443f-e086-4e1e-83bf-0f5a3d7d3e47 (blue)
    port blue1
        addresses: ["00:00:00:12:0b:01"]
    port blue2
        addresses: ["00:00:00:13:0b:01"]
air$
```

## References

Many thanks to all those docs!

- [OVN primar](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)
- [OVN tutorial](http://openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html)
- [OVN with Kubernetes](https://github.com/openvswitch/ovn-kubernetes/blob/master/README.md)

Happy Hacking!
