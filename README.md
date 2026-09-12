# cidr-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Subnet arithmetic: the questions you ask a network rather than an
address.  Is this prefix inside that one; what are the sixteen `/28`s
in this `/24`; what is the smallest set of prefixes covering exactly
these addresses and no others; which of my rules matches this packet.

It sits **on top of** [ipaddr-nv](https://github.com/novolang/ipaddr-nv)
and does not replace it.  ipaddr-nv has the address types, the network
types, the CIDR parsers and printers, `contains`, `overlaps`, the masks
and the host bounds.  This package is everything after that.

Five modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **arithmetic** | `ipnet` | you need a supernet, a subnet, or the nth host |
| the **set algebra** | `ipagg` | you have a pile of prefixes and want the minimal cover |
| the **registries** | `ipreg` | you are writing a route filter or a bogon list |
| the **names** | `iprdns` | you are generating a reverse zone |
| the **match** | `iptrie` | you have a table of prefixes and an address |

## Adding it, and checking it

```bash
novo pkg add cidr-nv           # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/ipnet_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: cidr-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use ip
use cidr
use ipnet
use iptrie

fn main() [io]
    // Which of my prefixes does this address belong to?
    let table = [cidr.net4(ip.ipv4(10, 0, 0, 0), 8).net,
                 cidr.net4(ip.ipv4(10, 1, 2, 0), 24).net]
    let t = iptrie.build4(table)
    println("${iptrie.lookup4(t, ip.ipv4(10, 1, 2, 3))}")   // 1, the /24

    // The second /28 inside a /24, without building the other fifteen.
    let n = cidr.net4(ip.ipv4(192, 0, 2, 0), 24).net
    println("${ipnet.nth_subnet4(n, 28, 1).net.network().octet(3)}")   // 16
```

## The load-bearing interface

`iptrie.lookup4` — and the fact that it answers an `Int`.

```novo norun:pseudo
pub fn lookup4(t: CidrTrie4, addr: Ipv4) -> Int []
```

A routing table's rows carry a next hop.  A firewall's carry an
action.  A geolocation table's carry a country, an ASN, a rate limit.
The one thing all of them have in common is a prefix, and the one
thing this package knows how to do is find which prefix is longest.
So the match answers an **index into the list of prefixes the caller
handed over**, and what lives at that position is none of this
package's business.

A trie parameterised over a payload type would have been the obvious
alternative, and it would have made the payload allocate — which the
device consumer cannot have, and which would have put a generic
parameter in every signature to buy a `list.at` the caller can write.

The second decision the rest follows from is that **there is no second
network type**.  Public type names are keyed by name across the whole
assembly, so a package declaring its own `Net4` would be a package
nobody could use alongside ipaddr-nv.  The plan's row said this one is
"the split if subnet arithmetic outgrows that module": the arithmetic
outgrew it, the types did not, so everything here takes and answers
ipaddr-nv's own values and a consumer adds this package only when they
need what it adds.

## The device claim, and what it is shaped by

`tests/embedded_probe.nv` is built for `--target=nrf52-qemu`, and what
it exercises is the device face of `iptrie`: `CidrRule4` — three `Int`s
in a `@value` struct — and the two predicates `rule_holds4` and
`rule_beats4`, over which the **caller** writes its own loop across its
own fixed array.

That face speaks raw integers rather than `Ipv4`, and both reasons are
facts rather than preferences:

- On a device the four octets come out of a packet header, not out of
  a string.  ipaddr-nv's parsers carry `@tier(app)` precisely because
  the embedded runtime defines no string-byte read — so the address a
  firmware holds was never an `Ipv4` to begin with.
- The audit builds a device probe against this package's own types
  only, so a device surface naming ipaddr-nv's would be a claim that
  could not be checked.

A table of thirty-two rules is 768 bytes of `.rodata` and the match is
a loop with two comparisons in it.  `rule_from_net4` is the bridge: the
host reads a configuration file into `Net4` values, converts, and
flashes the array.

There is no device trie, on purpose.  A firmware firewall has eight to
thirty-two rules and building a trie over them costs more than scanning
them.

## The three things that are easy to get wrong, and what this package does about each

**Aggregation must not round up.**  `ipagg.aggregate4` answers a set
holding every address the input held and no address it did not.  A
summariser allowed to round up is a summariser that produces a firewall
rule letting through an address nobody listed.  The rounding-up version
exists and is called `enclosing4`, with a different name and a doc
comment that says what it includes, so a reader has to decide which
they are doing.

**A predicate about a network is not a predicate about an address.**
`10.0.0.0/8` is private; `10.0.0.0/7` is half private and half public.
ipaddr-nv's `is_private` asks about an address and is right; an
`is_private` that took a network and answered on its first address
would say yes to the `/7`.  So `ipreg.is_private4` answers about
*every* address in the network, `partly_private4` answers about any of
them, and the two disagreeing is what an audit wants to be shown.

**Hosts and subnets are index arithmetic, never a list.**  A `/8` has
16 777 214 usable hosts and a `/64` has more addresses than a machine
has bytes.  `nth_host4(n, i)` and `host_count4(n)` are the whole
iteration API, and there is no `hosts()` returning a list because on
the networks people actually have it could not return.

`ipagg.range_to_nets4` is the one place a list is the right answer: the
minimal cover of an arbitrary range is at most 62 networks, and that
number is small enough to hand over.

## Counts that do not fit

An IPv6 `/32` has 2^96 addresses and `Int` is 64 bits.  Those counts
answer `-1`, which is the convention ipaddr-nv's `num_addresses`
already set.  `ipnet.host_bits6` is the answer that always fits, and it
is the one a caller deciding whether a prefix is worth enumerating
actually wants — "2^96" is a decision and `-1` is not.

## The registries are a value, not a file

`ipreg.entries4()` and `entries6()` hand over the IANA special-purpose
rows as data — about eighty of them between the two families — each
with the five flags the registry defines as separate fields.  They are
separate because no single label describes `100.64.0.0/10`, which is
forwardable and not globally reachable, and that combination is exactly
what a carrier-grade NAT needs.

A `core` package cannot fetch anything and should not want to: a table
downloaded at run time is a table whose contents differ between two
runs of the same firewall.  `ipreg.registry_date()` says how old the
compiled-in snapshot is, because a table inside a package goes stale
silently.

## The netmask form

ipaddr-nv **refuses** to parse `192.0.2.0/255.255.255.0`, for a stated
reason: the same four octets can be a netmask or a hostmask and the two
mean different prefixes, so a parser that accepted the form would have
to guess.  `iptrie.prefix_of_mask4` is the other half of that decision
— the conversion is available where a caller has already decided which
of the two they are holding, and it is not available by guessing inside
a parser.  It is also what a device configured with `255.255.255.0`
needs.

## Reverse DNS

`iprdns` builds the name and asks nobody; sending the query is dns-nv's
job.  The interesting part is not reversing four octets, it is which
zone a *prefix* belongs to: `in-addr.arpa` delegates on octet
boundaries only, so a `/24` has one zone, a `/23` has two, and a `/25`
has none of its own — it is a fragment of its parent's, served by the
RFC 2317 `CNAME` trick or not served at all.  `zones_for4` answers
that; `classless_zone4` is the trick, with its own name because the `/`
inside a label is legal in a zone name and illegal in a host name.

## Dependencies

One: **ipaddr-nv**, `core`.  See the load-bearing interface above for
why there is exactly one and why it is this one.

## Ports

[ipnetwork](https://github.com/achanda/ipnetwork) and Python's
[`ipaddress`](https://docs.python.org/3/library/ipaddress.html) network
half are the reference implementations; `ipaddress`'s
`collapse_addresses`, `address_exclude`, `subnet_of` and `supernet_of`
are ported by name, and its test suite is the oracle for the
aggregation.  The IANA
[IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry) and
[IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry)
special-purpose registries are the source of `ipreg`'s table.

## Licence

Apache-2.0.
