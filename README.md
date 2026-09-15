# cidr-nv

Classless Inter-Domain Routing (CIDR) is the way the Internet names a
block of addresses rather than one address. It is specified in
[RFC 4632](https://www.rfc-editor.org/rfc/rfc4632). This package brings
the arithmetic over those blocks to novo-lang: containment, splitting,
aggregation, the IANA special-purpose registries, reverse-DNS names and
longest-prefix match. It is built on
[ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv), which owns the
address and network types, and it declares no types of its own to
replace them.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What CIDR is

An IP address names one interface. A **prefix** names a block of them.
CIDR writes a prefix as an address, a slash and a **prefix length**:
`192.0.2.0/24` is the 256 addresses whose leading 24 bits are the
leading 24 bits of `192.0.2.0`. RFC 4632 section 3.1 defines the
notation. The bits the prefix length covers are the **network part**
and the rest are the **host part**. The address with every host bit
zero is the **network address**, and for IPv4 the address with every
host bit one is the **broadcast address**.

A shorter prefix holds more addresses. A **supernet** of a network is a
network with a shorter prefix that contains it, and a **subnet** is a
network with a longer prefix contained in it. The immediate supernet,
one bit shorter, is the **parent**. The other network with the same
parent is the **sibling**. Two siblings joined together are exactly
their parent, which is the single fact route **aggregation** is built
out of: RFC 4632 section 4 describes aggregation as the reason
classless addressing was adopted at all.

**Longest-prefix match** is the question a routing table, a firewall
and a geolocation table all ask. Several prefixes in a table may hold
one address, and the one that decides the outcome is the one with the
longest prefix.

A **range** is not a network. `10.0.0.5` through `10.0.0.9` is five
addresses, and no single prefix holds exactly those five. Converting a
range into the smallest set of prefixes covering exactly it is its own
operation, and the answer is a short list.

| Quantity | IPv4 | IPv6 |
| --- | --- | --- |
| Address width | 32 bits | 128 bits |
| Prefix lengths | 0 to 32 | 0 to 128 |
| Minimal cover of one arbitrary range | at most 62 networks | at most 254 networks |
| A device rule | 24 bytes | 32 bytes |

The IANA special-purpose address registries hold about eighty rows
between the two families. Each row is a prefix that is not ordinary
public address space, such as `10.0.0.0/8`, `127.0.0.0/8` or
`100.64.0.0/10`, together with the attributes that say how it may be
used. [RFC 6890](https://www.rfc-editor.org/rfc/rfc6890) defines those
attributes.

This package performs no input or output. It reads no file, asks no
resolver a question and consults no clock. Every function is arithmetic
over values the caller already holds, so the same code runs in a route
filter on a server, in a zone-file generator and in a firewall table on
a microcontroller.

## Install

```
novo pkg add cidr-nv
```

## Example

```novo
use ip
use cidr
use ipnet
use ipreg
use iptrie

fn main() [io]
    // The two prefixes this program routes on, in the order it will
    // read them back in.
    let corp = cidr.net4(ip.ipv4(10, 0, 0, 0), 8).net
    let branch = cidr.net4(ip.ipv4(10, 1, 2, 0), 24).net

    // Build the match structure once, over that list.
    let t = iptrie.build4([corp, branch])

    // The longest prefix holding this address, as a position in the
    // list above. It answers 1, because the /24 is longer than the /8.
    println("${iptrie.lookup4(t, ip.ipv4(10, 1, 2, 3))}")

    // True, because every address in the /8 is RFC 1918 private.
    println("${ipreg.is_private4(corp)}")

    // The second /28 inside a /24, without building the other fifteen.
    // Its network address is 192.0.2.16, so the last octet is 16.
    let n = cidr.net4(ip.ipv4(192, 0, 2, 0), 24).net
    println("${ipnet.nth_subnet4(n, 28, 1).net.network().octet(3)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: cidr-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ipnet` | The arithmetic on one network: subnet and supernet relations, the parent and the sibling, the subnets and the addresses by index rather than as a list, the counts, and the ordering. |
| `ipagg` | The arithmetic on a pile of networks: address ranges and their minimal cover, lossless aggregation, the shortest enclosing prefix, cutting a hole in a network, and difference, intersection and coverage. |
| `ipreg` | The IANA special-purpose registries as a table of rows, and predicates that answer about a whole network rather than one address. |
| `iprdns` | Reverse-DNS names in `in-addr.arpa` and `ip6.arpa` in both directions, and which zones a prefix is delegated as. |
| `iptrie` | Longest-prefix match, in two forms: a built table for a host, and a rule value with two predicates for a device. |

## How to choose an entry point

`ipnet` answers a question about one network. `ipagg` answers a
question about a set of them. If you are asking "is this prefix inside
that one" you want `ipnet`; if you are asking "what is the smallest set
of prefixes covering all of these" you want `ipagg`.

`iptrie` has two ways in, and they suit different callers.

**`build4` and `lookup4` are the host form.** `build4` takes a list of
networks and returns a structure that is queried many times. `lookup4`
answers the position in that list of the longest prefix holding an
address, or `-1`. Use it for a table with hundreds or thousands of rows
that is built once.

**`CidrRule4`, `rule_holds4` and `rule_beats4` are the device form.** A
rule is three integers: the masked network address, the prefix length,
and a tag the caller assigns and this package never reads. The two
predicates are the inside of a scan, and you write the loop over your
own array. Use it for the eight to thirty-two rules a firmware firewall
has, where building a table costs more than scanning one.

`rule_from_net4` and `net_from_rule4` convert between the two forms.
The ordinary arrangement is a host that reads a configuration file into
networks, converts them into rules, and flashes the array.

## The rules a user needs

1. **Aggregation is lossless and enclosure is not.** `ipagg.aggregate4`
   answers a set holding every address the input held and no address it
   did not. `ipagg.enclosing4` answers the single shortest prefix
   holding them all, which also holds addresses nobody listed:
   `10.0.0.0/24` and `10.0.2.0/24` enclose as `10.0.0.0/22`. Pick by
   name. RFC 4632 section 4 is the aggregation this implements.
2. **A predicate about a network is not a predicate about an address.**
   `ipreg.is_private4` is true only when *every* address in the network
   is private. `10.0.0.0/8` is private and `10.0.0.0/7` is not, because
   half of it is `11.0.0.0/8`, which is public.
   `ipreg.partly_private4` is the one that answers about any address in
   the network. RFC 1918 section 3 is the private space.
3. **Hosts and subnets are index arithmetic, never a list.** A `/8`
   holds 16 777 214 usable hosts and an IPv6 `/64` holds more addresses
   than a machine has bytes. `host_count4` with `nth_host4`, and
   `subnet_count4` with `nth_subnet4`, are the whole iteration surface.
   No function returns the hosts of a network.
4. **A `/31` holds two hosts and no broadcast address, and a `/32`
   holds one.** RFC 3021 defines the `/31` case, which exists because
   point-to-point links were wasting two addresses in four.
   `ipnet.host_count4` knows both cases; a subtraction does not.
5. **A count that needs more than 64 bits answers `-1`.** Every IPv6
   prefix shorter than 65 reaches this. `ipnet.host_bits6` answers
   `128 - prefix`, which always fits, and is what a caller deciding
   whether to enumerate a prefix actually wants. `-1` is the convention
   ipaddr-nv's `num_addresses` already set.
6. **A fallible answer carries an error code beside the value.** Call
   `is_ok()` before reading `.net`, `.addr` or `.range`. The language
   refuses a `@value` struct as a `Result` payload (SPEC section 14.5)
   and `Net4` is one, so `CidrNet4Ans` and its siblings are the shape
   ipaddr-nv already uses for the same reason.
7. **This package's error codes are numbered above ipaddr-nv's.**
   `ip.error_of` answers `None` for them rather than the wrong reason,
   and `ipnet.reason` is what names them. The two vocabularies do not
   overlap.
8. **`iptrie.lookup4` answers an index into the list you passed to
   `build4`.** It never answers a value of its own, and `-1` means no
   prefix held the address. Whatever you stored beside each prefix is
   yours to look up.
9. **A duplicate prefix is reported, not repaired.** `lookup4` answers
   the first of two identical prefixes, and `duplicates4` lists the
   later ones. On the device side, `rule_beats4` is false for an equal
   prefix length, so a scan in table order keeps the first of two
   equally specific rules.
10. **A device rule must have its base masked.** A rule with host bits
    set matches nothing and looks correct. `iptrie.rule4` masks and
    `rule_is_normal4` checks. An unused slot in a fixed table is
    `rule_none4()`, whose prefix is `-1`, because prefix 0 would match
    everything and prefix 32 with base 0 would match `0.0.0.0`.
11. **A dotted netmask is converted, never parsed.** ipaddr-nv refuses
    `192.0.2.0/255.255.255.0`, because the same four octets can be a
    netmask or a hostmask and the two mean different prefixes.
    `iptrie.prefix_of_mask4` does the conversion where a caller already
    knows which of the two they hold, and answers `-1` when the mask is
    not a run of ones followed by a run of zeros.
12. **Reverse zones are delegated on an octet boundary for IPv4 and a
    nibble boundary for IPv6.** A `/24` has one zone, a `/23` has two
    and a `/25` has none of its own. `iprdns.zones_for4` answers the
    empty list for a prefix that is not on a boundary, and
    `is_on_zone_boundary4` is how you tell that apart from a mistake.
    `classless_zone4` is RFC 2317's name for a prefix longer than a
    `/24`, and the `/` inside its first label is legal in a zone name
    and illegal in a host name. RFC 1035 section 3.5 defines
    `in-addr.arpa` and RFC 3596 section 2.5 defines `ip6.arpa`.
13. **An IPv6 reverse name is always thirty-two nibbles.** It is never
    abbreviated, because the zone is defined over the full form and an
    abbreviated name resolves to nothing. `iprdns.name6_len` is the
    length, which is the same for every address.
14. **The registry table is compiled into the package.** A `core`
    package fetches nothing, and a table downloaded at run time is a
    table whose contents differ between two runs of the same firewall.
    `ipreg.registry_date` answers the date of the snapshot, because a
    compiled-in table goes stale silently.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `iptrie`'s device form: `CidrRule4`,
`CidrRule6`, the two constructors, the mask arithmetic, the match
predicates and `rule_table_bytes4`. All of it is integer arithmetic
over values the caller owns.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program. It holds a fixed
rule array and the scan a firmware firewall writes over it, and it does
not build today. A device build compiles every module in the package
rather than only the ones the probe reaches, and `ipnet`'s four answer
structs cannot be value structs on a device because their fields are
`Net4`, `Net6`, `Ipv4` and `Ipv6`. Nothing yet checks the device build
automatically, so the claim rests on the code.

The device form speaks raw integers rather than `Ipv4` and `Ipv6`
values. On a device the four octets come out of a packet header rather
than out of a string, and ipaddr-nv's parsers are host-only for that
reason: the embedded runtime defines no string-byte read. A table of
thirty-two IPv4 rules is 768 bytes of read-only data, and
`rule_table_bytes4` is the function that says so.

The other four modules stay on the host. `ipagg`, `ipreg` and `iprdns`
each build a list or read a string, and `iptrie`'s host form builds
arrays.

## What is not included

- **Address and network types.** `Ipv4`, `Ipv6`, `Net4` and `Net6` are
  [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv)'s, and this
  package neither redeclares nor wraps them. A public type name is
  keyed by name across a whole program, so a second `Net4` on the
  registry would be a type nobody could use alongside the first.
- **Parsing and printing.** Reading `192.0.2.0/24` and writing it back
  out, including RFC 5952 for IPv6, is ipaddr-nv's work. That package
  also owns `contains`, `overlaps`, the masks and the host bounds.
- **A list of the hosts in a network.** See rule 3. Index arithmetic is
  the whole iteration surface.
- **A device trie.** A firmware firewall has eight to thirty-two rules,
  and building a trie over them costs more than scanning them.
- **Sending a DNS query.** `iprdns` builds the name and asks nobody.
  Putting a name on the wire needs a resolver, which no `core` package
  can be.
- **Fetching the IANA registries.** The table is compiled in. See
  rule 14.
- **An IPv6 counterpart for every IPv4 function.** `partly_private4`,
  `is_loopback4`, `is_shared4`, `total_addresses4`, `host_count4`,
  `nth_host4`, `index_of4` beyond 63 bits, `covering4` and
  `duplicates4` have no `6` twin. Each is either meaningless for IPv6
  or waiting for a caller who needs it, and adding one is a minor
  version rather than a breaking change.

## Related packages

- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) is the package
  most readers want first, and it is the one this package is built on.
  **It is implemented and working today; this one is not.** ipaddr-nv
  is about an address and about one network: parsing and formatting
  `192.0.2.1` and `2001:db8::/32`, RFC 5952 output, classifying a
  single address as private, loopback or documentation, `contains`,
  `overlaps`, the netmask and hostmask, the first and last host, and
  the address count. cidr-nv is about the arithmetic between networks:
  whether one prefix is inside another, the nth subnet or host without
  building the others, reducing a pile of prefixes to a minimal cover,
  classifying a whole prefix rather than one address, reverse-DNS zone
  names, and longest-prefix match over a table. Install ipaddr-nv
  alone if you need to read, print, compare or test addresses. Add
  cidr-nv on top of it only when you need one of the operations listed
  above, and only once version 0.1.0 has made them work.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) is the
  RFC 1035 wire format. It is what puts a name `iprdns` built into a
  query, and what reads the answer back.
- `std.net` in the standard library is sockets. It has no notion of a
  prefix, and nothing in this package touches it.

## Tests

```bash
novo test tests                      # 54 tests across the five modules
```

The vectors are Python's
[`ipaddress`](https://docs.python.org/3/library/ipaddress.html) test
suite, which is the reference for `subnet_of`, `supernet_of`,
`collapse_addresses` and `address_exclude`, and the Rust crate
[ipnetwork](https://github.com/achanda/ipnetwork). The registry rows
are the IANA
[IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry) and
[IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry)
special-purpose registries. Where this package differs from
`ipaddress` on purpose, such as the `/31` and `/32` host counts and the
direction check on `supernet4`, the difference is asserted rather than
inherited.

The suite checks that a network is a subnet of itself, that containment
is about every address rather than the first, that siblings collapse
and non-siblings do not, that enclosure rounds up where aggregation
does not, that a count that does not fit says so, that a reverse name
parses back to the address it names, that a `/25` is not on a zone
boundary, that a longer prefix wins a lookup, and that a duplicate
prefix is reported rather than repaired.

The tests compile today and fail at run, each on the
`not implemented: cidr-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `ipnet`: the containment relations, the supernets, the subnets and addresses by index, the counts and the ordering | no |
| `ipagg`: ranges, the minimal cover, aggregation, enclosure, exclusion and the set algebra | no |
| `ipreg`: the registry table, the whole-network predicates and the bogon lists | no |
| `iprdns`: the names, the parsers, the zones and the RFC 2317 delegation | no |
| `iptrie`, host form: `build4`, `build6`, `lookup4`, `lookup6`, `lookup_all4`, `lookup_all6`, `covering4`, `count4`, `count6`, `duplicates4` | no |
| `iptrie`, device form: `rule4`, `rule6`, `mask4`, `prefix_of_mask4`, `rule_holds4`, `rule_holds6`, `rule_beats4`, `rule_beats6`, `rule_is_normal4`, `rule_none4`, `rule_none6`, `rule_is_none4`, `rule_is_none6`, `rule_table_bytes4` | no |
| `iptrie`, the bridge: `rule_from_net4`, `rule_from_net6`, `net_from_rule4`, `net_from_rule6` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
