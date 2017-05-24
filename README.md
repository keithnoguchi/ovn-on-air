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
3. [configure.yml](#configureyml)

### build.yml

[build.yml](build.yml) is to setup the build environment on hv11, as explained in
[Documentation/intro/install/debian.rst](https://github.com/openvswitch/ovs/blob/master/Documentation/intro/install/debian.rst).

```
air$ ansible-playbook build.yml
```

This playbook will

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

### configure.yml

[configure.yml](configure.yml) will configure OvS/OVN for virtual networking.

```
air$ ansible-playbook configure.yml
```

## References

Many thanks to all those docs!

- [OVN primar](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)
- [OVN tutorial](http://openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html)
- [OVN with Kubernetes](https://github.com/openvswitch/ovn-kubernetes/blob/master/README.md)

Happy Hacking!
