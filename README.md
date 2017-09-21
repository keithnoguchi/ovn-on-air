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
4. [guest.yml](#guestyml)
5. [configure.yml](#configureyml)

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
air$ ssh hv11 sudo ovn-nbctl show
switch 8dcafba8-2782-4f0c-a328-77d8de53f542 (green)
    port green1
        addresses: ["00:00:00:12:0c:01"]
    port green4
        addresses: ["00:00:00:13:0c:04"]
    port green3
        addresses: ["00:00:00:12:0c:03"]
    port green2
        addresses: ["00:00:00:13:0c:02"]
switch b7566b94-4ada-4ed0-983a-2a0ef97e3daa (blue)
    port blue3
        addresses: ["00:00:00:12:0b:03"]
    port blue2
        addresses: ["00:00:00:13:0b:02"]
    port blue4
        addresses: ["00:00:00:13:0b:04"]
    port blue1
        addresses: ["00:00:00:12:0b:01"]
switch b445c4b7-6315-440f-b7df-af231a3c3903 (red)
    port red3
        addresses: ["00:00:00:12:0a:03"]
    port red4
        addresses: ["00:00:00:13:0a:04"]
    port red2
        addresses: ["00:00:00:13:0a:02"]
    port red1
        addresses: ["00:00:00:12:0a:01"]
air$
```

and the south bound database:

```
air$ ssh hv11 sudo ovn-sbctl show
Chassis "408b15fb-9d9e-42e8-aa7a-bb9bbbeb5ed2"
    hostname: "hv13"
    Encap geneve
        ip: "192.168.122.113"
        options: {csum="true"}
    Port_Binding "blue2"
    Port_Binding "green4"
    Port_Binding "blue4"
    Port_Binding "red2"
    Port_Binding "red4"
    Port_Binding "green2"
Chassis "fcd1baa6-0954-4440-a8e4-7170036ace15"
    hostname: "hv12"
    Encap geneve
        ip: "192.168.122.112"
        options: {csum="true"}
    Port_Binding "blue3"
    Port_Binding "blue1"
    Port_Binding "green3"
    Port_Binding "red3"
    Port_Binding "red1"
    Port_Binding "green1"
air$
```

Now, you can send ping between two guests on a separate
chassis.  Here is the ping result between `red1` to `red2`:

```
air$ ssh hv12 'sudo ip netns exec red ping -c5 10.0.1.2'
PING 10.0.1.2 (10.0.1.2) 56(84) bytes of data.
64 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=0.329 ms
64 bytes from 10.0.1.2: icmp_seq=2 ttl=64 time=0.550 ms
64 bytes from 10.0.1.2: icmp_seq=3 ttl=64 time=0.631 ms
64 bytes from 10.0.1.2: icmp_seq=4 ttl=64 time=0.614 ms
64 bytes from 10.0.1.2: icmp_seq=5 ttl=64 time=0.374 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3999ms
rtt min/avg/max/mdev = 0.329/0.499/0.631/0.127 ms
air$
```

And also, there is no reachability between `red1` and `blue2`:

```
air$ ssh hv12 'sudo ip netns exec red ping -c5 10.0.2.2'
^CKilled by signal 2.
air$
```

nor between `red1` and `blue1`:

```
air$ ssh hv12 'sudo ip netns exec red ping -c5 10.0.2.1'
^CKilled by signal 2.
air$
```

## References

Many thanks to all those docs!

- [OVN primer](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)
- [OVN tutorial](http://openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html)
- [OVN with Kubernetes](https://github.com/openvswitch/ovn-kubernetes/blob/master/README.md)

Happy Hacking!
