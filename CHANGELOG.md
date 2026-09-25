# Changelog

All notable changes to cidr-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves ipaddr-nv 0.1.2 to 0.1.4.  ipaddr-nv 0.1.2 writes
  into lists through names that are not declared `var`, which novo 0.11
  refuses (E2038), so this package did not build with novo 0.11 against
  it.  No requirement in the manifest changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ipnet` — `subnet_of`, `supernet_of`, `supernet`, `parent`,
  `sibling`; subnets and hosts by index rather than as lists;
  `host_bits6`, the count that always fits; and `CidrNet4Ans`, the
  error-code-beside-the-value shape ipaddr-nv already uses because
  SPEC § 14.5 refuses a `@value` struct as a `Result` payload.
- `ipagg` — `CidrRange4`, `range_to_nets4`, `aggregate4` (lossless),
  `enclosing4` (rounds up, and says so), `exclude4`, and the set
  algebra `difference`, `intersection` and `covered_by`.
- `ipreg` — the IANA special-purpose registries as a table of
  `CidrSpecial` rows with five separate flags, and predicates that
  answer about EVERY address in a network, with `partly_*` beside each
  for any of them.
- `iprdns` — `name4`, `name6` and their parsers; `zones_for4`,
  `zone_for6`, `is_on_zone_boundary4` and RFC 2317's `classless_zone4`.
- `iptrie` — `build4` / `lookup4` for the host, and `CidrRule4` with
  `rule_holds4` / `rule_beats4` for the device, over which the caller
  writes its own loop.

### Known

- **No second network type.** `Net4`, `Net6`, `Ipv4` and `Ipv6` are
  ipaddr-nv's throughout. The plan's row anticipated "the split if
  subnet arithmetic outgrows that module": the arithmetic outgrew it,
  the types did not.
- **The match answers an index** into the caller's own list, so a
  routing table, a firewall and a geolocation lookup all use it with
  no generic parameter and no allocation.
- **The device claim is built**, not asserted: `tests/embedded_probe.nv`
  links for `--target=nrf52-qemu` and runs the loop a firmware firewall
  writes.
- **The device face speaks raw integers**, because a firmware holds
  four octets out of a packet header rather than a parsed address, and
  because a probe may name only this package's own types.
- **Aggregation is lossless**; `enclosing4` is the rounding-up version
  and carries a different name for that reason.
- **A predicate about a network is not one about an address**:
  `10.0.0.0/7` is half private, and `is_private4` and `partly_private4`
  disagree about it on purpose.
- **Hosts and subnets are index arithmetic**, never a list.
  `range_to_nets4` is the one bounded list and it returns at most 62
  networks.
- **Counts that need more than 64 bits answer `-1`**, matching
  ipaddr-nv's existing convention; `host_bits6` is the exact answer.
- **The registry table is compiled in**, with `registry_date()` beside
  it, because a `core` package cannot fetch and a table that goes stale
  silently is worse than one that says its age.
- **The netmask form is converted but never parsed** — ipaddr-nv
  refuses it for a stated reason and `prefix_of_mask4` is the other
  half of that decision.
- **One `core` dependency**, ipaddr-nv.
