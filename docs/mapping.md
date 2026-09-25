# NetBox → AVD → Nexus as Code Field Mapping

This is the field-by-field reference for the thesis this project demonstrates: **one
vendor-neutral model in NetBox drives two different vendor renderers.** `nb-playbooks` reads
NetBox once and writes a neutral model to `build/dc1-model.yml` / `build/dc2-model.yml`.
`avd-playbooks` and `nac-playbooks` each turn that same shape of data into their own vendor
dialect — `eos_designs` for Arista AVD, a `vxlan:` service model for Cisco Nexus as Code (NaC).

See [`data-model.md`](data-model.md) for how each NetBox object is structured and why. This
document is about where each piece of that data ends up on each side.

## How to read this table

- **NetBox** — the object/field as modeled in NetBox.
- **AVD** — the `eos_designs` key it becomes, consumed by `arista.avd.eos_designs` /
  `eos_cli_config_gen`.
- **Nexus as Code** — the `vxlan:` key it becomes, consumed by `cisco.nac_dc_vxlan.validate`.
- *(unused)* means that side has no analogue and the field is silently ignored — not a bug,
  just a case where one tool needs something the other doesn't.

## Fabric-wide settings

| NetBox | AVD | Nexus as Code |
|---|---|---|
| `dcim.Site.name` | `fabric_name` (also the inventory group name) | `fabric.name` (also the inventory hostname and the `host_vars/<name>/` directory `validate` scans) |
| `ipam.ASN` assigned to site | `bgp_as` (every node) | `global.ibgp.bgp_asn` |
| Config context `underlay: ospf` | `underlay_routing_protocol` | *(implied — NaC's VXLAN EVPN template is OSPF underlay by default)* |
| Config context `overlay: ibgp` | `overlay_routing_protocol` | `global.ibgp.*` (the whole overlay block is keyed under `ibgp`) |
| Config context `mgmt_gateway` | `mgmt_gateway` | *(unused — NDFC discovery IP, not modeled here)* |
| Config context `dns_servers` / `ntp_servers` | `dns_settings.servers` / `ntp_settings.servers` | `global.ibgp.dns_servers` / `.ntp_servers` |
| Config context `l2vni_range` / `l3vni_range` | *(unused — AVD derives VNIs from `vni_override`/`vrf_vni` per object, no range needed)* | `global.ibgp.layer2_vni_range` / `layer3_vni_range` (`{from, to}`) |
| `VLANGroup.vid_ranges` | *(unused)* | `global.ibgp.layer2_vlan_range` / `layer3_vlan_range` |
| Config context `vpc.peer_link_vlan`, `keepalive_vrf`, `auto_recovery_time`, `peer_link_po` | *(AVD derives MLAG peer-link settings from cabling instead — see below)* | `global.ibgp.vpc.{peer_link_vlan, peer_keep_alive, auto_recovery_time, peer_link_port_channel_id}` |
| count(role=spine) | *(implied — spines are just nodes of type `spine`)* | `global.ibgp.route_reflectors` (a count, not a device list) |

## Devices

| NetBox | AVD | Nexus as Code |
|---|---|---|
| `Device.name` | inventory hostname | `switches[].hostname` |
| `Device.primary_ip4` | `mgmt_ip` | `switches[].mgmt_ip_address` |
| `Device.serial` | *(unused)* | **`switches[].serial_number`** — mandatory; validation fails without it |
| `DeviceRole` | `node_type_keys` mapping (`spine`→`spine`, `leaf`→`l3leaf`) | `switches[].role` (`spine` / `leaf`) |
| CF `node_id` | **`nodes[].id`** — required by AVD, drives pool offsets | *(unused — no stable-integer identity needed on this side)* |
| CF `mlag_domain_id` (shared by a pair) | `node_groups[].name` grouping + `mlag: true` (2 nodes in a group auto-enables MLAG) | `vpc_peers[].domain_id`, with `peer1`/`peer2` the two hostnames |
| Cables between MLAG peers | `node_groups[].nodes[].mlag_interfaces` (peer-link ports) | `vpc_peers[].peer_link_interfaces` |
| Anycast IP on peers' `Loopback1` | *(unused — AVD derives the VTEP VIP itself for an MLAG pair)* | **`vpc_peers[].vtep_vip`** |

## Cabling → topology

Cables are never hand-listed in either renderer — both derive uplinks/fabric links by walking
the cable graph `nb-playbooks` already read.

| NetBox | AVD | Nexus as Code |
|---|---|---|
| Leaf↔spine cables | `uplink_switches` / `uplink_interfaces` / `uplink_switch_interfaces` (per node) | `topology.fabric_links` (a flat list of endpoint pairs) |
| Prefix role `loopback-routerid` | `loopback_ipv4_pool` | `underlay.ipv4.loopback0_ip_range` |
| Prefix role `loopback-vtep` | `vtep_loopback_ipv4_pool` | `underlay.ipv4.loopback1_ip_range` |
| Prefix role `underlay-p2p` | `uplink_ipv4_pool` | `underlay.ipv4.subnet_range` |
| Prefix role `mlag-peer-l2` / `mlag-peer-l3` | `mlag_peer_ipv4_pool` / `mlag_peer_l3_ipv4_pool` | *(NaC derives the vPC peer-link subnet itself)* |

Actual per-device IPs are **not** stored in NetBox — pools are the shared intent, and AVD and
NDFC each allocate from them independently. They will differ between the two sides, and that is
correct: it's a renderer's job to allocate, not the model's.

## Tenants, VRFs, and networks

| NetBox | AVD | Nexus as Code |
|---|---|---|
| `tenancy.Tenant` | `tenants[].name` | *(implied — NaC has no separate tenant grouping construct; VRFs stand alone)* |
| `ipam.VRF.name` | `tenants[].vrfs[].name` | `overlay.vrfs[].name` |
| CF `vni` on VRF | `tenants[].vrfs[].vrf_vni` | `overlay.vrfs[].vrf_id` |
| `ipam.VRF.rd` | *(deliberately left `null` — see below)* | *(deliberately left `null` — see below)* |
| `ipam.VLAN.name` | `svis[].name` | `overlay.networks[].name` / `.vlan_name` |
| `ipam.VLAN.vid` | `svis[].id` | `overlay.networks[].vlan_id` |
| CF `vni` on VLAN | `svis[].vni_override` | `overlay.networks[].net_id` |
| Service prefix (VRF+VLAN), `.1` | `svis[].ip_address_virtual` | `overlay.networks[].gw_ip_address` |
| Tag `svc-tenant1` on VLAN | `svis[].tags` + node `filter.tags` (per-device membership) | **synthesized** `network_attach_groups` / `vrf_attach_groups` — see below |
| L3VNI placeholder VLAN (Cisco-only) | *(unused — symmetric IRB needs no per-VRF transport SVI in AVD)* | `overlay.vrfs[].vrf_vlan_id` |

### The attach-group synthesis

This is the biggest structural difference between the two renderers, and it's worth calling out
explicitly rather than burying it in a table row: **AVD expresses service membership per
device** (each leaf's `svis` list says which VLANs that leaf carries), while **NaC expresses it
through named groups** (`network_attach_groups`/`vrf_attach_groups`, each a named list of
switches). NetBox stores device↔service membership as tags; `nac_inputs` builds the groups with
a single `selectattr` over the device list — there's no group concept in NetBox to keep in sync.

### The RD is deliberately not stored

`ipam.VRF.rd` is left `null` in NetBox on purpose, and the two renderers do different things
with that:

- **AVD** doesn't need an `rd_override` at all — its actual default (`overlay_rd_type
  .admin_subfield` = `router_id`) computes `<device's own Loopback0 IP>:<vrf_vni>`, giving every
  leaf its own unique, correct RD with zero input. Confirmed against the live DC1 deploy
  (`rd 10.0.0.5:50001` on `dc1-leaf1` — a per-device value, not one shared across the fabric).
- **Nexus as Code**'s offline `validate` produces no `rd` key anywhere in the golden or extended
  service model. RD assignment for NDFC-managed fabrics happens at actual deploy time against a
  live controller, so this project — which only validates, never deploys, on the Cisco side —
  can't observe or confirm that behavior. Documented here as a known gap rather than guessed at.

## What's genuinely one-sided

| Field | Side | Why |
|---|---|---|
| `Device.serial` | NaC only | NDFC discovers/manages switches by serial; AVD has no equivalent requirement |
| CF `node_id` | AVD only | `eos_designs` mandates a stable per-node integer `id` for pool offsetting; NaC has no analogous need |
| `vpc_peers[].vtep_vip` (anycast IP) | NaC only | AVD derives the MLAG pair's VTEP VIP itself; NaC needs it as explicit input |
| `network_attach_groups` / `vrf_attach_groups` | NaC only | AVD's per-device `svis`/`filter.tags` already express the same membership without a group indirection |
| VRF/RD | Effectively neither | See above — both sides compute or defer it rather than taking it from NetBox |
