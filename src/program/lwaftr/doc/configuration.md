# Configuration

The lwAFTR's configuration is modelled by a
[YANG](https://tools.ietf.org/html/rfc6020) schema,
[snabb-softwire-v1](../../../lib/yang/snabb-softwire-v1.yang).

The lwAFTR takes its configuration from the user in the form of a text
file.  That file's grammar is derived from the YANG schema; see the
[Snabb YANG README](../../../lib/yang/README.md) for full details.
Here's an example:

```
softwire-config {
  // IPv4 interface.
  external-interface {
    allow-incoming-icmp true;
    // Limit ICMP error transmit rate to PACKETS/PERIOD.
    error-rate-limiting {
      packets 600000;
      period 2;
    }
    // Generate ICMP errors at all?
    generate-icmp-errors true;
    // Basic parameters.
    ip 10.10.10.10;
    mac 12:12:12:12:12:12;
    mtu 1460;
    // vlan-tag 42;
    // Where to go next.  Either one will suffice; if you specify the IP,
    // the next-hop MAC will be determined by ARP.
    next-hop {
      mac 68:68:68:68:68:68;
      ip 1.2.3.4;
    }
    // Control the size of the fragment reassembly buffer.
    reassembly {
      max-fragments-per-packet 40;
      max-packets 20000;
    }
  }
  // The same thing as for the external interface, but on the IPv6 side
  // and with IPv6 addresses.
  internal-interface {
    allow-incoming-icmp true;
    error-rate-limiting {
      packets 600000;
      period 2;
    }
    generate-icmp-errors true;
    // One more interesting thing -- here we control whether to support
    // routing traffic between two B4s.
    hairpinning true;
    ip 8:9:a:b:c:d:e:f;
    mac 22:22:22:22:22:22;
    mtu 1500;
    // vlan-tag 64;
    next-hop {
      mac 44:44:44:44:44:44;
      // NDP instead of ARP of course.
      ip 7:8:9:a:b:c:d:e;
    }
    reassembly {
      max-fragments-per-packet 40;
      max-packets 20000;
    }
  }
  // Now the binding table!  3 parts: PSID map, BR address table, and
  // softwire set.  See description below for details.
  binding-table {
    psid-map { addr 178.79.150.15; psid-length 4; }
    psid-map { addr 178.79.150.233; psid-length 16; }
    psid-map { addr 178.79.150.2; psid-length 16; }
    br-address 8:9:a:b:c:d:e:f;
    br-address 1e:1:1:1:1:1:1:af;
    br-address 1e:2:2:2:2:2:2:af;
    softwire {
      ipv4 178.79.150.233;
      psid 22788;
      b4-ipv6 127:11:12:13:14:15:16:128;
    }
    softwire {
      ipv4 178.79.150.233;
      psid 2700;
      b4-ipv6 127:11:12:13:14:15:16:128;
    }
    softwire {
      ipv4 178.79.150.15;
      psid 1;
      b4-ipv6 127:22:33:44:55:66:77:128;
    }
    softwire {
      ipv4 178.79.150.2;
      psid 7850;
      b4-ipv6 127:24:35:46:57:68:79:128;
      br 1;
    }
  }
}
```

Basically there's an `external-interface` section defining the
parameters around the IPv4 interface that communicates with the
internet, an `internal-interface` section doing the same for the IPv6
side that communicates with the B4s, and then the `binding-table` that
declares the set of softwires.  The whole thing is surrounded in a
`softwire-config { ... }` block.

## Compiling conigurations

When a lwAFTR is started, it will automatically compile its
configuration if it does not find a compiled configuration that's fresh.
However for large configurations with millions of binding table entries,
this can take a second or two, so it can still be useful to compile the
configuration ahead of time.

Use the `snabb lwaftr compile-configuration` command to compile a
configuration ahead of time.  If you do this, you can use the `snabb
lwaftr control PID reload` command to tell the Snabb process with the
given *PID* to reload the table.

## In-depth configuration explanation

See the embedded descriptions in the
[snabb-softwire-v1](../../../lib/yang/snabb-softwire-v1.yang) schema
file.

## Binding tables

A binding table is a collection of softwires (tunnels).  One endpoint
of the softwire is in the AFTR and the other is in the B4.  A
softwire provisions an IPv4 address (or a part of an IPv4 address) to
a customer behind a B4.  The B4 arranges for all IPv4 traffic to be
encapsulated in IPv6 and sent to the AFTR; the AFTR does the reverse.
The binding table is how the AFTR knows which B4 is associated with
an incoming packet.

In the Snabb lwAFTR there are three parts of a binding table: the PSID
info map, the border router (BR) address table, and the softwire map.

### PSID info map

The PSID info map defines the set of IPv4 addresses that are provisioned
by an lwAFTR.  It also defines the way in which those addresses are
shared, by specifying the "psid-length" and "shift" parameters for each
address.  See RFC 7597 for more details on the PSID scheme for how to
share IPv4 addresses.  IPv4 addresses are added to the `psid-map` with
statements like this:

```
  psid-map {
    addr 178.79.150.3;
    end-addr 178.100.150.3;
    psid-length 6;
    reserved-ports-bit-count 0;
    shift 10;
  }
```

`end-addr`, `reserved-ports-bit-count`, and `shift` are all optional.  

An entry's `psid-length`, `shift` and `reserved-ports-bit-count`
parameters must necessarily add up to 16, so it is sufficient to specify
just two of them.  Actually it's sufficient to just specify the
`psid-length`, which is the only required one.
`reserved-ports-bit-count` defaults to 0, and `shift` defaults to `16 -
psid-length`.

### Border router addresses

Next, the `br-address` clauses define the set of IPv6 addresses to
associate with the lwAFTR.  These are the "border router" addresses.
For a usual deployment there will be one main address and possibly some
additional ones.  For example:

```
  br-address 8:9:a:b:c:d:e:f;
  br-address 1E:1:1:1:1:1:1:af;
  ...
```

### Softwires

Finally, the `softwire` clauses define the set of softwires to
provision.  Each softwire associates an IPv4 address, a PSID, and a B4
address.  For example:

```
  softwire { ipv4 178.79.150.233; psid 80; b4-ipv6 127:2:3:4:5:6:7:128; }
```

By default, a softwire is associated with the first `br-address`
(`br 0;`).  To associate the tunnel with a different border router,
specify it by index:

```
  softwire { ipv4 178.79.150.233; psid 80; b4-ipv6 127:2:3:4:5:6:7:128; aftr 0; }
```

## Ingress and egress filters

Both the `internal-interface` and `external-interface` configuration
blocks support `ingress-filter` and `egress-filter` options.
```
...
ingress-filter "ip";
egress-filter "ip";
```

If given these filters should be a
[pflang](https://github.com/Igalia/pflua/blob/master/doc/pflang.md)
filter.  Pflang is the language of `tcpdump`, `libpcap`, and other
tools.

If an ingress or egress filter is specified in the configuration file,
then only packets which match that filter will be allowed in or out of
the lwAFTR.  It might help to think of the filter as being "whitelists"
-- they pass only what matches and reject other things.  To make a
"blacklist" filter, use the `not` pflang operator:

```
// Reject IPv6.
ingress-filter "not ip6";
```

You might need to use parentheses so that you are applying the `not` to
the right subexpression.  Note also that if you have 802.1Q vlan tagging
enabled, the ingress and egress filters run after the tags have been
stripped.

Here is a more complicated example:

```
egress-filter "
  ip6 and not (
    (icmp6 and
     src net 3ffe:501:0:1001::2/128 and
     dst net 3ffe:507:0:1:200:86ff:fe05:8000/116)
    or
    (ip6 and udp and
     src net 3ffe:500::/28 and
     dst net 3ffe:0501:4819::/64 and
     src portrange 2397-2399 and
     dst port 53)
  )
";
```

Enabling ingress and egress filters currently has a performance cost.
See [performance.md](performance.md).
