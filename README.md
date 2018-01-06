# OVN on air

Let's make [OVN, Open Virtual Network](http://openvswitch.org/support/dist-docs/ovn-architecture.7.html),
on [MacBook Air](https://github.com/keinohguchi/arch-on-air/blob/master/README.md)!

1. [Setup](#setup)
2. [Playbooks](#playbooks)

## Setup

I'm using KVM/libvirt to have those three KVM instances on Air and
reachable through `hv12`, `node20`, and `node21`, respectively.  And
to make the playbook clean, I'm using ubuntu 16.04 for all three
KVM instances.

```
                      +------------+
                      |            |
                      |    hv12    |
                      |            |
                      +------+-----+
                             |
                             |
                             |
         +------------+      |     +------------+
         |            |      |     |            |
         |    node20  |      |     |    node21  |
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
4. [guest.yml](#guestyml)
5. [configure.yml](#configureyml)

### build.yml

[build.yml](build.yml) is to setup the build environment on hv12, as explained in
[Documentation/intro/install/debian.rst](https://github.com/openvswitch/ovs/blob/master/Documentation/intro/install/debian.rst).

```
air$ ansible-playbook build.yml
```

This playbook will:

1. setup the `hv12` as a build machine
2. git clone the latest OvS from the github
3. build OvS/OVN
4. download the deb package from the build machine to the local `/tmp` directory

### install.yml

Before installing your greatest OvS/OVN, better to remove the default OvS/OVN
packages, to avoid the confusion, especially you installed those in a different
locations, e.g. `/usr/local/...`.  You can do that by `sudo apt remove`, if you
installed those through the standard APT package manager.

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
air0$ ssh hv12 netstat -antp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:6641            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:6642            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:51478         127.0.0.1:4369          ESTABLISHED -
tcp        0    236 192.168.122.112:22      192.168.122.1:49548     ESTABLISHED -
tcp        0      0 192.168.122.112:4369    192.168.122.112:33081   TIME_WAIT   -
tcp        0      0 127.0.0.1:4369          127.0.0.1:51478         ESTABLISHED -
tcp        0      0 192.168.122.112:6642    192.168.122.121:56008   ESTABLISHED -
tcp        0      0 192.168.122.112:6642    192.168.122.120:47662   ESTABLISHED -
tcp6       0      0 :::5672                 :::*                    LISTEN      -
tcp6       0      0 :::4369                 :::*                    LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
air0$
```

### guest.yml

Now, all the hosts are properly configured and connected each other,
let's take care of the guests next with [guest.yml](guest.yml).
As explained by [Dustin Spinhirne](http://blog.spinhirne) in his
[wonderful OVN primer](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html),
we'll use Linux name space to quickly simulate the guests.

```
air$ ansible-playbook guest.yml
```

This playbook will create name space based guests, aka fake guests, with the
following steps:

1. Create name spaces
2. Create itnerfaces through the `ovs-vsctl`
3. Move those interfaces to the appropriate name space
4. Assign the MAC address
5. Assign the IP address
6. Set the interface state to up

### configure.yml

Now, all the hosts and guests configured and up and running, let's connects
those together by creating the virtual network through OVN through
[configure.yml](configure.yml)!

```
air$ ansible-playbook configure.yml
```

This playbook will:

1. Create logical switches for each tenant, e.g. red, blue, green
2. Create logical ports, e.g. red1, red2, under the appropriate switches
3. Setup MAC address and MAC address based port security on the appropriate switches
4. Map the physical ports to the logical ports to connect the dots!

Here is the result of the logical switches in north bound database
after the playbook run:

```
air0$ ssh hv12 sudo ovn-nbctl show
switch baff58cc-649e-47da-9423-14bd20a7de0d (red)
    port red4
        addresses: ["00:00:00:13:0a:04"]
    port red1
        addresses: ["00:00:00:12:0a:01"]
    port red2
        addresses: ["00:00:00:13:0a:02"]
    port red3
        addresses: ["00:00:00:12:0a:03"]
switch 9aac643d-a00c-43a0-a139-27548a3f073a (green)
    port green4
        addresses: ["00:00:00:13:0c:04"]
    port green1
        addresses: ["00:00:00:12:0c:01"]
    port green3
        addresses: ["00:00:00:12:0c:03"]
    port green2
        addresses: ["00:00:00:13:0c:02"]
switch b153ee2a-00c5-45c0-88e9-e1566f929bde (blue)
    port blue1
        addresses: ["00:00:00:12:0b:01"]
    port blue2
        addresses: ["00:00:00:13:0b:02"]
    port blue4
        addresses: ["00:00:00:13:0b:04"]
    port blue3
        addresses: ["00:00:00:12:0b:03"]
air0$
```

and the south bound database:

```
air0$ ssh hv12 sudo ovn-sbctl show
Chassis "849d82f5-c492-4839-a81d-6d5d68e78c37"
    hostname: "node20.local"
    Encap geneve
        ip: "192.168.122.120"
        options: {csum="true"}
    Port_Binding "red3"
    Port_Binding "green3"
    Port_Binding "blue1"
    Port_Binding "blue3"
    Port_Binding "green1"
    Port_Binding "red1"
Chassis "2d91afc5-32dd-4dbd-a958-679fdb268a2d"
    hostname: "node21.local"
    Encap geneve
        ip: "192.168.122.121"
        options: {csum="true"}
    Port_Binding "red2"
    Port_Binding "blue2"
    Port_Binding "blue4"
    Port_Binding "green2"
    Port_Binding "green4"
    Port_Binding "red4"
air0$
```

Now, you can send ping between two guests on a separate
chassis.  Here is the ping result between `red1` to `red2`:

```
air0$ ssh node20 sudo ip netns exec red ping -c5 10.0.1.2
PING 10.0.1.2 (10.0.1.2) 56(84) bytes of data.
64 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=1.38 ms
64 bytes from 10.0.1.2: icmp_seq=2 ttl=64 time=0.693 ms
64 bytes from 10.0.1.2: icmp_seq=3 ttl=64 time=0.720 ms
64 bytes from 10.0.1.2: icmp_seq=4 ttl=64 time=0.733 ms
64 bytes from 10.0.1.2: icmp_seq=5 ttl=64 time=0.674 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4001ms
rtt min/avg/max/mdev = 0.674/0.840/1.383/0.273 ms
air0$
```

And also, there is no reachability between `red1` and `blue2`:

```
air$ ssh node20 'sudo ip netns exec red ping -c5 10.0.2.2'
^CKilled by signal 2.
air$
```

nor between `red1` and `blue1`:

```
air$ ssh node20 'sudo ip netns exec red ping -c5 10.0.2.1'
^CKilled by signal 2.
air$
```

## References

Many thanks to all those docs!

- [OVN primer](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)
- [OVN tutorial](http://openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html)
- [OVN with Kubernetes](https://github.com/openvswitch/ovn-kubernetes/blob/master/README.md)

Happy Hacking!
