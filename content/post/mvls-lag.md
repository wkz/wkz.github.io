---
title: "Marvell LinkStreet LAG Issues"
date: 2022-05-17T15:15:48+02:00
---

While running some kselftest-like tests on a system made up of three
Marvell LinkStreet chips, I uncovered a series of issues related to
offloaded link aggregates (LAGs) that are spread across multiple
chips.

The system has the following layout:

```
                    .--0-1-2-3-4--.
              .-----a     sw1     |
        .-0-1-4-.   '--5-6-7-8-9--'   sw0: 6353 (Agate)
CPU +---6  sw0  |                     sw1: 6097 (Opal+)
        '-2-3-5-'   .--0-1-2-3-4--.   sw2: 6097 (Opal+)
              '-----a     sw2     |
                    '--5-6-7-8-9--'
```

Software-wise, the system is running
[NetBox](https://github.com/westermo/netbox) with 5.18 Linux kernel,
using the `mv88e6xxx` driver to control the switch chips.


# Inconsistent Hashing

After creating a LAG consisting of `sw0p1` and `sw1p1`, I was running
a test where multicast was to be flooded out through the LAG. During
that test, I observed that some groups were correctly flooded, while
others were not.

If a packet was forced out of the LAG port{{< sidenote >}}By injecting
a `FROM_CPU` packet on the CPU port{{< /sidenote >}}, it was correctly
received on the other side. So I knew that the problem was in the
forwarding plane of the switch.

Previously, I had run the same test, but with `sw1p0` and `sw2p4` as
the LAG ports, without any issues. Guessing that the issue might have
something to do with asymmetric hashing, I generated some test packets
with [nemesis](https://github.com/libnet/nemesis):

```shell
for i in $(seq 0 255); do
	nemesis ethernet -c 1 -d eth0 -M 01:00:de:ad:00:$(printf '%2.2x' $i) \
		-T 0xbbbb -P <(echo testing $i)
done
```

On the receiver I observed the following results:

| # Groups | # Copies | Interpretation                                                       |
|---------:|---------:|:---------------------------------------------------------------------|
|       64 |        0 | Both switches assume that the designated port is on the other switch |
|      128 |        1 | Switches are in agreement                                            |
|       64 |        2 | Both switches determines that the local port is the designated one   |

Disabling the hashing{{< sidenote >}}By clearing
`Global1:Reg7:Bit11`{{< /sidenote >}} causes the devices to fall back
to a simple XOR-based port selection. In this mode of operation,
sending the same test packets, exactly one copy of each of the 256
packets is received.

**Conclusion**: The Agate uses a different hash function than the one
on the Opal+.


## Suggested Solution

~~Opportunistically enable hashing until a configuration is encountered
which prevents it. As a first step, you could disable hashing as soon
as a cross-chip LAG is detected.~~

~~To support even more cases, you could also keep hashing enabled as
long as all cross-chip LAGs are setup between compatible chips. This
would require some more information from Marvell though, i.e. the
groups of chips using the same hash function.~~

**UPDATE:** It turns out that this issue is a side-effect of a silicon
bug in the Agate, for which an easy workaround exists. An undocumented
field{{< sidenote>}}`Global1/Reg10/Bit0-1`{{< /sidenote >}} has an
incorrect value when the Agate comes out of reset.

In more recent chips of the same family, this field specifies the hash
mode to use for ATU bucket selection, where `1` is "Default" and `3`
is "Direct" (`0` and `2` are reserved).

The Agate has a reset value of `0` whereas Peridot and Amethyst both
reset to `1`. Setting the the value to `1` resolves the issue.

Thus, it appears that the field in question, in addition to
influencing the hash function used for ATU bucket selection, also
affects the hash function used for LAG member selection.


# DSA Tag Trunk Bit Override

Because of the way `mv88e6xxx` handles port isolation{{< sidenote>}}Full
disclosure: This is my fault{{< /sidenote >}}, frames assigned to VID 0
ingressing on DSA ports are trapped to the CPU using the VTU policy feature.

This avoids a whole slew of issues where intermediate switches may be
confused by looking up the DA in the ATU. Unfortunately it also means
that the original DSA tag, which looked something like this...

```
FORWARD dev:2 port:0 trunk:yes
```

...is rewritten to:

```
TO_CPU  dev:2 port:0 code:policy-trap
```

Since there is not trunk bit in the `TO_CPU` tag, the original source
information is lost.

Fortunately, this is only an issue for LAGs in standalone mode - as
soon as a LAG is added to a bridge, no packets will ever be assigned
to VID 0. If a standalone LAG is required, you can create one using a
mode that is not possible to offload{{< sidenote>}}E.g. `balance-rr`
for Bond interfaces{{< /sidenote >}}. In that case, the DSA layer in
the kernel will fallback to a software LAG and everything will work as
expected.

**Conclusion**: The VTU policy feature can't coexist with offloaded
standalone LAGs.

## Suggested Solution

When a LAG is created, accept the offload in the `mv88e6xxx` driver if
it is supported, but do not actually push the configuration down to
hardware until the LAG is attached to a bridge. Before that point, the
"offloading" does very little anyway, the reason you want it offloaded
is so that you can hardware switch packets in and out of it - in
standalone mode all packets have to pass through the CPU anyway.


# ATU Trunk Bit Inheritance

When `mv88e6xxx` adds a static FDB entry to the ATU, it will reuse any
existing ATU entry for the DA in question{{< sidenote >}}This is most
likely because it makes the code reusable for MDB operations{{< /sidenote>}}.

Unfortunately, the Trunk bit of the ATU entry is not cleared before
entry is written back with the new port information.

**Example**: We start with the following bridge setup:

```
  br0
  / \
 / lag0  lag1
'   /\    /\
0  1  2  3  4
   |  '--'  |
   '--------'
```

Then we send a packet from `lag1`, which is physically looped back to
`lag0`, where it is received and learned by the ATU.

Now let's say that we swap the roles of the LAGs, i.e. we connect
`lag1` to the bridge and keep `lag0` as a standalone interface.

At this point, the bridge will want to add a static entry for the MAC
address of `lag1`, pointing towards the CPU port. It will then find
the existing dynamic entry from the previous configuration, override
the port and state and the write it back to the ATU. But since the
trunk bit is not cleared, you now end up with a static entry pointing
towards a non-existing LAG `0x400` (for the common case where the CPU
port is 11).

**Conclusion**: The trunk bit must always be cleared when updating an
existing ATU entry.

## Suggested Solution

Do that.
