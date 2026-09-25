# NetBox Data Model

This document describes what's built in NetBox for this project, and why each piece is modeled
the way it is. DC1 (Arista) and DC2 (Cisco) both exist, built from the identical pattern
described here -- DC2's own object values are called out inline wherever they differ from DC1's.

## The idea in one sentence

NetBox holds one vendor-neutral description of the fabric. Arista AVD and Cisco Nexus as Code
each read it and render their own dialect from it. Nothing vendor-specific lives in NetBox
except two fields on each device (`device_type`, `platform`) — everything else in this document
is the same regardless of which renderer eventually reads it.

## Design principle: native objects first

NetBox ships a rich native schema. Before adding a custom field or reaching for a plugin, ask
whether an existing object already means the same thing. Three examples from this model:

- A **config context** (built into NetBox core) replaces what an earlier version of this project
  did with a custom object type — see [Config contexts](#config-contexts-not-custom-objects)
  below.
- A **VLAN group's `vid_ranges`** carries what Nexus as Code calls
  `layer2_vlan_range`/`layer3_vlan_range` — no separate object needed.
- **Sharing a value in a plain integer custom field** (`mlag_domain_id`) is enough to express an
  MLAG/vPC pair — no need for `dcim.VirtualChassis` or a custom object type. See
  [MLAG/vPC pairing](#mlagvpc-pairing) below for why the more obvious-looking option
  (`VirtualChassis`) is actually wrong here.

## Site = Fabric

Each fabric is one NetBox site. DC1 (`dc1`) is the Arista half of the demo and is deployed live
via containerlab. DC2 will be the Cisco half, rendered only (no live deployment).

| Object | Value |
|---|---|
| Site | `dc1` |
| Rack | `dc1-r01` (19", 42U) |

## Devices

Six devices, one device type, one interface template set shared by all of them:

| Device | Role | Position | Serial | Management IP | `node_id` | `mlag_domain_id` |
|---|---|---|---|---|---|---|
| `dc1-spine1` | spine | 40 | DEMO0000001 | 172.20.21.11/24 | 1 | — |
| `dc1-spine2` | spine | 39 | DEMO0000002 | 172.20.21.12/24 | 2 | — |
| `dc1-leaf1` | leaf | 4 | DEMO0000003 | 172.20.21.13/24 | 3 | 1 |
| `dc1-leaf2` | leaf | 3 | DEMO0000004 | 172.20.21.14/24 | 4 | 1 |
| `dc1-leaf3` | leaf | 2 | DEMO0000005 | 172.20.21.15/24 | 5 | — |
| `dc1-leaf4` | leaf | 1 | DEMO0000006 | 172.20.21.16/24 | 6 | — |

`dc1-leaf1`/`dc1-leaf2` are an MLAG pair (shared `mlag_domain_id: 1`). `dc1-leaf3`/`dc1-leaf4`
are standalone VTEPs today; `dc1-leaf4` is the intended future border leaf (see
[Adding a border leaf](#adding-a-border-leaf-later) below).

Serial numbers are synthetic (`DEMO000000N`) — the model shape is real, the data is fake. Nexus
as Code mandates a serial number per switch (its validator treats a missing one as a HIGH
severity error); AVD never reads this field. This is a deliberate asymmetry, not an oversight.

Two roles exist but are unassigned so far: `border` and `edge-router`. They're created now,
alongside `spine`/`leaf`, specifically so a later phase can assign them without a schema change.

### Device type: one type, reused across projects

Manufacturer `Arista`, platform `ceos`, and device type `cEOS` all already existed in this
NetBox instance from a prior project (`netbox_configure`/`avd-poc`) — that device type had zero
interface templates defined, so this project extended it rather than creating a near-duplicate
`cEOS-lab` type. All 6 devices share it, which means all 6 devices also share the same 11
interface templates, auto-instantiated by NetBox on device creation:

`Management0` (1000base-t, mgmt-only), `Ethernet1`–`Ethernet8` (10gbase-x-sfpp),
`Loopback0`/`Loopback1` (virtual).

No device in this project has its own hand-maintained interface list — that's the payoff of
putting the interfaces on the device type instead of the device.

## Cables: the entire topology, in one place

Ten cables *are* the DC1 topology — nothing else describes what's connected to what. Every
renderer derives uplinks and peer-links by reading the cable graph back out; nothing is ever
hand-listed a second time elsewhere.

**Underlay (8 links, full mesh leaf↔spine):**

| Leaf | Ethernet1 → | Ethernet2 → |
|---|---|---|
| `dc1-leaf1` | `dc1-spine1` Ethernet1 | `dc1-spine2` Ethernet1 |
| `dc1-leaf2` | `dc1-spine1` Ethernet2 | `dc1-spine2` Ethernet2 |
| `dc1-leaf3` | `dc1-spine1` Ethernet3 | `dc1-spine2` Ethernet3 |
| `dc1-leaf4` | `dc1-spine1` Ethernet4 | `dc1-spine2` Ethernet4 |

**Peer-link (2 links, MLAG pair only):** `dc1-leaf1` Ethernet7/8 ↔ `dc1-leaf2` Ethernet7/8,
bundled into `Port-Channel1` on both peers.

Ethernet5/6 are deliberately left free on every leaf for future endpoints or a border-leaf
uplink.

## IP addressing: pools are the intent, addresses are the output

This is the one place where AVD and Nexus as Code fundamentally disagree, and the model is built
around that disagreement rather than papering over it: **AVD computes every device address
deterministically from a pool; NDFC allocates addresses from a pool at deploy time.** They will
produce different actual IPs from the same pool, every time, and that's correct — not a bug to
fix. So NetBox stores the pools (the shared intent) and nothing else, with two exceptions where
a NetBox-recorded address is unavoidable.

### Prefix pools (`is_pool: true`, scoped to site `dc1`)

| Prefix | Role | Purpose |
|---|---|---|
| `172.20.21.0/24` | `management` | Device management network — must match the containerlab mgmt subnet exactly |
| `10.0.0.0/24` | `loopback-routerid` | Router-ID loopback (Loopback0) |
| `10.0.1.0/24` | `vtep-loopback` | VTEP loopback (Loopback1) |
| `10.1.0.0/24` | `underlay-p2p` | Point-to-point underlay uplinks |
| `10.255.252.0/24` | `mlag-peer-l2` | MLAG/vPC peer-link, L2 |
| `10.255.253.0/24` | `mlag-peer-l3` | MLAG/vPC peer-link, L3 |

`management`, `vtep-loopback`, and `underlay-p2p` are prefix roles that already existed in this
NetBox instance from the prior project and are reused by name — same underlying concept, no
reason to invent parallel names for it.

**Why `172.20.21.0/24` and not `.20.0/24`:** the existing `avd-poc` project already runs a live
containerlab topology on this same host with its management network pinned to
`172.20.20.0/24`. Reusing that subnet would collide with a live Docker network. `.21.0/24` is
DC1's own space; `clab/dc1-arista.clab.yml` (written in a later phase) must use it.

### The two addresses that *are* stored

- **Management IPs** (one per device, `172.20.21.11`–`.16`) — these have to be real, fixed
  addresses because containerlab needs to assign them and Ansible needs to reach the devices.
- **The MLAG/vPC VTEP VIP** (`10.0.1.101/32` at DC1, `10.2.1.101/32` at DC2, both role
  `Anycast`) — Nexus as Code's `vpc_peers[].vtep_vip` is a mandatory field with no equivalent AVD
  needs, so it can't be computed by a renderer; it has to be recorded. It's modeled as a single
  **unassigned** IP address rather than the same address attached to both leaf peers' Loopback1
  — NetBox's IP address module has no clean idempotent way to represent "one address, two
  interfaces" (re-running would either silently move it or create a duplicate). An unassigned
  VIP sidesteps that, and arguably models the concept more accurately: the VIP belongs to the
  *pair*, not to either individual interface. Since it's unassigned, there's no interface to
  scope it to a site by — the export instead matches "the Anycast IP inside this site's own
  vtep-loopback pool" (`?parent=<pool>`), which is how a second site's VIP stays distinguishable
  from the first without adding a dedicated NetBox field just for that.

## Overlay: VLANs, VRF, VNIs

| Object | DC1 | DC2 |
|---|---|---|
| VLAN groups | `dc1-l2vni` (100–199), `dc1-l3vni` (2000–2099) | `dc2-l2vni` (100–199), `dc2-l3vni` (2000–2099) |
| VLAN `TENANT1-APP` | vid 110, VNI 10110 | vid 110, VNI 10110 |
| VLAN `TENANT1-WEB` | vid 120, VNI 10120 | vid 120, VNI 10120 |
| VLAN `TENANT1-L3VNI` | vid 2001 — transport VLAN for the VRF's SVI; carries no VNI of its own | vid 2001 |
| VRF `TENANT1` | one shared object — VNI 50001 | (same object) |
| Service prefixes | `10.10.110.0/24`, `10.10.120.0/24` | `10.20.110.0/24`, `10.20.120.0/24` |

Same vids and VNIs at both sites, deliberately — DC1 and DC2 are two independent fabrics, not
one stretched across sites, but reusing identical service numbers demonstrates the same model
producing the same service definition on both vendor stacks.

The anycast gateway for each service prefix is the first usable address (`.1`) by convention —
there's no separate "gateway" field to keep in sync.

**The VRF's RD is deliberately not stored.** It was hardcoded as `65001:50001` (DC1's ASN) in an
earlier version of this model — copying that straight to DC2 (ASN 65002) would have produced a
VRF with the wrong site's ASN baked into its route distinguisher, a real correctness bug a
network engineer would notice. Both AVD and Nexus as Code derive the RD correctly from each
site's own ASN by default, so NetBox just leaves the field unset (`rd: null` — not `""`, which
NetBox normalizes back to `null` on read anyway, which made an earlier attempt at "clearing" it
look changed on every single run).

The VLAN group `vid_ranges` (100–199, 2000–2099) are directly what Nexus as Code calls
`global.ibgp.layer2_vlan_range`/`layer3_vlan_range`. A native NetBox object is carrying settings
that an earlier design would have needed a custom object type for.

## Custom fields — exactly three

Everything that couldn't be expressed with a native field, and nothing more:

| Field | Applies to | Type | Why it exists |
|---|---|---|---|
| `node_id` | `dcim.device` | integer | AVD requires a stable per-device identity (`nodes[].id`) to derive pool offsets. No native NetBox field is a safe fit — rack position is a physical U location, not an identity, and the database primary key isn't something to build automation around. Nexus as Code doesn't use this at all. |
| `mlag_domain_id` | `dcim.device` | integer | Devices that share a value are an MLAG (AVD)/vPC (Nexus as Code) pair. Also doubles directly as the vPC domain ID. |
| `vni` | `ipam.vlan` **and** `ipam.vrf` | integer | Nexus as Code requires and range-validates an explicit VNI on every VLAN (L2VNI) and VRF (L3VNI); AVD can derive one but accepts an override. One field, applied to two object types, covers both cases — see below for why this isn't `ipam.L2VPN` instead. |

**Why not `ipam.L2VPN` for VNIs?** NetBox has a native VXLAN-EVPN L2VPN object that looks like
the obvious fit for a VNI. It was considered and rejected: an `L2VPNTermination` can only attach
to a VLAN, interface, or VM interface — never a VRF — so it has no way to carry the L3VNI. Using
`L2VPN` for the L2VNI and a custom field for the L3VNI would mean two different mechanisms for
the same concept, which fails the project's own "if it's hard to explain, it's a bad idea"
standard. A single symmetric custom field covers both cases the same way.

## Config contexts, not custom objects

Fabric-wide settings — underlay/overlay protocol, replication mode, anycast gateway MAC, MTU,
VNI ranges, vPC timers, DNS/NTP — are **not** custom fields and **not** a custom object type.
They're native NetBox config contexts, layered by weight:

| Context | Weight | Scope | Carries |
|---|---|---|---|
| `fabric-common` | 1000 | global | `underlay: ospf`, `overlay: ibgp`, `replication: ingress`, `anycast_gateway_mac`, `p2p_mtu: 9214`, `l2vni_range: "10000-19999"`, `l3vni_range: "50000-59999"`, vPC timers (`peer_link_po`, `peer_link_vlan`, `keepalive_vrf`, `auto_recovery_time`) |
| `dc1-fabric` | 1100 | site `dc1` | `fabric_name: DC1`, `mgmt_gateway: 172.20.21.1`, `dns_servers`, `ntp_servers` |
| `dc2-fabric` | 1100 | site `dc2` | `fabric_name: DC2`, `mgmt_gateway: 172.20.22.1`, `dns_servers`, `ntp_servers` |

NetBox merges these automatically by weight — a device sees `fabric-common` plus its own site's
context combined, with the site-scoped layer taking precedence on any overlapping key. Adding
DC2 changed nothing about `fabric-common`; it only added one more site-scoped layer alongside
`dc1-fabric`, exactly as designed. The same `anycast_gateway_mac` is shared by both fabrics
deliberately — they're independent, non-interconnected fabrics, so there's no requirement for
distinct virtual-router MACs, and reusing one is simpler.

**Why this instead of a custom object type:** an earlier version of this kind of model
(`netbox_configure`'s `avd_fabric_data` custom object type) needed a third-party plugin, a
hand-defined schema, and manual code to resolve relationships to sites/roles by name. None of
that is actually needed here — this data is a bag of settings that should be visible to a device
based on where it lives, which is exactly what a config context is built for. It also merges by
weight automatically, is read natively by the `nb_inventory` Ansible plugin
(`config_context: true`), and needs no plugin at all.

## MLAG/vPC pairing

`dcim.VirtualChassis` looks like the obvious way to represent an MLAG/vPC pair in NetBox, and
it's the wrong tool: a virtual chassis models one logical device with one control plane, but an
MLAG/vPC pair is two independent control planes that happen to share state. Concretely, the
`nb_inventory` Ansible plugin reassigns a virtual chassis member's interfaces to the chassis
master — which would make `dc1-leaf2`'s cables disappear from inventory and break every
renderer's uplink derivation.

Instead, `dc1-leaf1` and `dc1-leaf2` simply share the same `mlag_domain_id` value. A pair is
"devices with the same value" — `peer1`/`peer2` fall out of sorting by name, peer-link ports
come from the cable graph (see above), and the VTEP VIP is the anycast IP address described
above. One integer field does the whole job.

## Adding a border leaf later

`border` and `edge-router` device roles already exist, unassigned. Turning `dc1-leaf4` into a
border leaf is meant to be additive, not a rework:

1. Change `dc1-leaf4`'s role to `border`.
2. Add an `edge-router` device and one cable to it.
3. Add VRF-lite settings to a new, higher-weight config context scoped to the `border` role.

No VRF, VLAN, tag, pool, or cable data already in the model needs to move — a border leaf is
still a VTEP leaf in both AVD and Nexus as Code; it just carries one more role and a bit more
config context data.

## Mapping to each renderer

The full field-by-field mapping (NetBox → AVD `eos_designs` → Nexus as Code `vxlan:`) will be
documented in `docs/mapping.md` once the render layer exists (`avd-playbooks`/`nac-playbooks`).
The short version: every native object above has a direct AVD input and a direct Nexus as Code
input except where noted — `node_id` is AVD-only, `serial` is Nexus-as-Code-only, and the vPC
VTEP VIP has no AVD equivalent because AVD derives it instead of needing it recorded.
