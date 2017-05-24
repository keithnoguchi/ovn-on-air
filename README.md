# OVN on air

Let's make OVN, Open Virtual Networking, on MacBook Air!

## Setup

I'm using KVM/libvirt to have those three KVM instances on Air and
reachable through `hv11`, `hv12`, and `hv13`, respectively:

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

## References

Many thanks to all those docs!

- [OVN primar](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)
- [OVN tutorial](http://openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html)
- [OVN with Kubernetes](https://github.com/openvswitch/ovn-kubernetes/blob/master/README.md)

Happy Hacking!
