# IPv6 IOAM: Linux Kernel implementation

IPv6 IOAM is available in the Linux kernel since version 5.15. The support for
in-transit traffic (ip6ip6 encapsulation) is also available since version 5.16.

IOAM can be configured with the iproute2 tool (IOAM support is also available).

Right now, the implementation only supports the Pre-allocated Trace Option.

## Configuration example

![Topology](./topology.png?raw=true "Topology")

For this example, an IOAM domain will be configured from **Athos** to **Aramis**,
but not on the reverse path.

Here is the IOAM basic configuration for **Athos** with the corresponding
commands:

```
+===================================================================+
|                    Athos - IOAM configuration                     |
+===================================================================+
| Node ID                     | 1                                   |
+-------------------------------------------------------------------+
| Node Wide ID                | 11111111                            |
+-------------------------------------------------------------------+
| Ingress ID                  | 101                                 |
+-------------------------------------------------------------------+
| Ingress Wide ID             | 101101                              |
+-------------------------------------------------------------------+
| Egress ID                   | 102                                 |
+-------------------------------------------------------------------+
| Egress Wide ID              | 102102                              |
+-------------------------------------------------------------------+

$ sysctl -wq net.ipv6.ioam6_id=1
$ sysctl -wq net.ipv6.ioam6_id_wide=11111111
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id=101
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id_wide=101101
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id=102
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id_wide=102102

+-------------------------------------------------------------------+
| Namespace (=123) Data       | 0xdeadbee0                          |
+-------------------------------------------------------------------+
| Namespace (=123) Wide Data  | 0xcafec0caf00dc0de                  |
+-------------------------------------------------------------------+
| Schema ID (Namespace=123)   | 777                                 |
+-------------------------------------------------------------------+
| Schema Data (Namespace=123) | something that will be 4n-aligned   |
+-------------------------------------------------------------------+

$ ip ioam namespace add 123 data 0xdeadbee0 wide 0xcafec0caf00dc0de
$ ip ioam schema add 777 "something that will be 4n-aligned"
$ ip ioam namespace set 123 schema 777
```

Here is the IOAM configuration for **Porthos** with the corresponding commands:

```
+=================================================================+
|                  Porthos - IOAM configuration                   |
+=================================================================+
| Node ID                     | 2                                 |
+-----------------------------------------------------------------+
| Node Wide ID                | 22222222                          |
+-----------------------------------------------------------------+
| Ingress ID                  | 201                               |
+-----------------------------------------------------------------+
| Ingress Wide ID             | 201201                            |
+-----------------------------------------------------------------+
| Egress ID                   | 202                               |
+-----------------------------------------------------------------+
| Egress Wide ID              | 202202                            |
+-----------------------------------------------------------------+

$ sysctl -wq net.ipv6.ioam6_id=2
$ sysctl -wq net.ipv6.ioam6_id_wide=22222222
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id=201
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id_wide=201201
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id=202
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id_wide=202202

+-----------------------------------------------------------------+
| Namespace (=123) Data       | 0xdeadbee1                        |
+-----------------------------------------------------------------+
| Namespace (=123) Wide Data  | 0xcafec0caf11dc0de                |
+-----------------------------------------------------------------+
| Schema ID (Namespace=123)   | 666                               |
+-----------------------------------------------------------------+
| Schema Data (Namespace=123) | Hello there -Obi                  |
+-----------------------------------------------------------------+

$ ip ioam namespace add 123 data 0xdeadbee1 wide 0xcafec0caf11dc0de
$ ip ioam schema add 666 "Hello there -Obi"
$ ip ioam namespace set 123 schema 666
```

Here is the IOAM configuration for **Aramis** with the corresponding commands:

```
+==================================================================+
|                   Aramis - IOAM configuration                    |
+==================================================================+
| Node ID                    | 3                                   |
+------------------------------------------------------------------+
| Node Wide ID               | 33333333                            |
+------------------------------------------------------------------+
| Ingress ID                 | 301                                 |
+------------------------------------------------------------------+
| Ingress Wide ID            | 301301                              |
+------------------------------------------------------------------+
| Egress ID                  | 302                                 |
+------------------------------------------------------------------+
| Egress Wide ID             | 302302                              |
+------------------------------------------------------------------+

$ sysctl -wq net.ipv6.ioam6_id=3
$ sysctl -wq net.ipv6.ioam6_id_wide=33333333
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id=301
$ sysctl -wq net.ipv6.conf.eth0.ioam6_id_wide=301301
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id=302
$ sysctl -wq net.ipv6.conf.eth1.ioam6_id_wide=302302

+------------------------------------------------------------------+
| Namespace (=123) Data      | 0xdeadbee2                          |
+------------------------------------------------------------------+
| Namespace (=123) Wide Data | 0xcafec0caf22dc0de                  |
+------------------------------------------------------------------+
| Schema ID                  | None                                |
+------------------------------------------------------------------+
| Schema Data                |                                     |
+------------------------------------------------------------------+

$ ip ioam namespace add 123 data 0xdeadbee2 wide 0xcafec0caf22dc0de
```

As mentioned, the IOAM domain is enabled from **Athos** to **Aramis** but not on
the reverse path, which means that IOAM is only allowed on ingress for both
**Porthos** and **Aramis** (not on **Athos** since it is the ingress of the IOAM
domain and no IOAM data is supposed to enter). The corresponding configuration
is as follows:

```
Porthos$ sysctl -wq net.ipv6.conf.eth0.ioam6_enabled=1
Aramis$ sysctl -wq net.ipv6.conf.eth0.ioam6_enabled=1
```

If **Alpha** (the sender) wants to send traffic to **Beta** (the receiver),
packets will go through the IOAM domain where **Athos** will insert an IOAM
option (Pre-allocated Trace) inside a Hop-by-hop. The IOAM insertion must be
enabled on **Athos** before compiling its kernel:

```
CONFIG_IPV6_IOAM6_LWTUNNEL=y
```

*RFC8200* does not allow direct insertion for in-transit traffic. Therefore, an
ip6ip6 encapsulation is applied to carry IOAM data (the tunnel destination will
be **Aramis**, to prevent IOAM data from leaking the domain). In this example,
we will only require the first bit of the trace type (hop_limit+node_id) but you
could select whatever trace type you want as long as you adapt the "size"
parameter (which is, basically, the pre-allocated size in bytes as a 4-byte
multiple). Note that you could also use another mode (inline, encap, auto)
depending on your use case. For this use case, the corresponding configuration
for **Athos** is as follows:

```
$ ip -6 route add db03::/64 encap ioam6 mode encap tundst db02::2 trace prealloc type 0x800000 ns 123 size 12 dev eth1
```

As a result, **Athos** encapsulates IOAM data inside an outter IPv6 header,
leading to an IPv6-in-IPv6 packet. Therefore, IPv6 tunnel decapsulation must be
enabled on **Aramis** with the following commands:

```
modprobe ip6_tunnel
ip link set ip6tnl0 up
```

Note that the `modprobe` command is only required if `ip6_tunnel` was compiled as
a module (which is the most common case).

That's it, you are now ready to run your own IOAM domain !
