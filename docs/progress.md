# Progress Log

## 2026-09-21 — NetBox model loader for DC1 (Phase 2 complete)

Built and ran `nb-playbooks/load-model.yml` against the live NetBox at `blnetbox.lab.local`
(shared with the `netbox_configure`/`avd-poc` project — same instance, new site, no collisions).
Ten roles, one per object type, in dependency order: `custom_fields`, `tenancy`, `sites`,
`device_types`, `device_roles`, `ipam`, `overlay`, `devices`, `cables`, `contexts`. Var files
live in `nb-playbooks/inventory/host_vars/netbox/` as numbered files (`00-custom-fields.yml`
through `80-contexts.yml`), one per role.

**Deviations from the original design, all convention-over-configuration calls made after
checking the live instance rather than guessing:**
- Reused the existing `Arista` manufacturer, `ceos` platform, and `cEOS` device type (not new
  `eos`/`cEOS-lab`) — it already existed with zero interface templates, so extending it was
  simpler than creating a near-duplicate. Added the 11 interface templates
  (`Management0`, `Ethernet1`-`8`, `Loopback0`/`1`) to it.
- Reused the existing private RIR `Bob Co RIR` for ASN 65001 instead of creating an `RFC6996`
  RIR — no RFC6996 RIR existed, and this instance already has its own private-RIR convention.
- Reused existing prefix roles `management`, `vtep-loopback`, `underlay-p2p` by name; only
  `loopback-routerid`, `mlag-peer-l2`, `mlag-peer-l3` were newly created.
- DC1's management subnet is `172.20.21.0/24`, not `172.20.20.0/24` — the latter is already a
  live Docker network (`clab-mgmt`) for the existing `avd-poc` containerlab topology on this
  same host. `clab/dc1-arista.clab.yml` (not yet written) must use `172.20.21.0/24` to match.
- The MLAG/vPC VTEP VIP (`10.0.1.101/32`, role Anycast) is modeled as a single **unassigned**
  IP address, not the same address assigned to both peers' Loopback1. NetBox's
  `netbox_ip_address` module has no clean idempotent way to represent "same address on two
  interfaces" (`state: present` would just reuse/move the one record; `state: new` would create
  a fresh duplicate on every run). An unassigned VIP is simpler and arguably more correct
  modeling anyway — the VIP belongs to the pair, not to either individual interface.
- `ipam.ASN` has no `site` field and NetBox 4.2+ removed `dcim.Site.asn` in favor of a
  `Site.asns` many-to-many relation that no `netbox.netbox` module (3.23.0) exposes yet. The
  `ipam` role has one small `ansible.builtin.uri` GET+PATCH pair to link them — the only place
  this project drops to a raw API call instead of a module.

**Idempotency:** ran the playbook twice; second run reported zero changes except where a
transient NetBox connection blip mid-run on the *first* attempt had left the 4 leaf devices
not-yet-created (the 2 spines succeeded before the blip) — a re-run picked them up cleanly.
Not a defect in the model.

Both linters clean throughout (`.ansible-lint` needed one addition: `args[module]` is suppressed
with a comment, since ansible-lint checks roles in isolation and can't see the play-level
`module_defaults` that actually satisfies `netbox_url`/`netbox_token` at runtime).

Credentials: `nb-playbooks/inventory/group_vars/all/vault.yml` is a copy of
`netbox_configure`'s vaulted `servers.yml` (same `netbox_api`/`netbox_token` vars, still
encrypted — reused intentionally, not duplicated by accident). `.vault_pass` was copied from
`capitalone-avd-poc/avd-playbooks/.vault_pass` and is gitignored.

### Next up — Phase 3: neutral export
Build `nb-playbooks/export-model.yml`: read DC1 back out of NetBox (`nb_inventory` or direct API
calls) and write `build/fabric-model.yml` (the vendor-neutral model both renderers will read)
plus `build/dc1-inventory.yml` (plain YAML Ansible inventory, `ansible_host` = mgmt IP without
the `/24`). Remember the known `nb_inventory` wart: it strips `primary_ip4` to a bare address,
but AVD's `mgmt_ip` wants the CIDR mask, so an extra lookup is needed for that field specifically.
Agree the exported schema before writing either renderer, since both `avd-playbooks` and
`nac-playbooks` depend on it and neither should touch NetBox directly.

## 2026-09-17 — Planning + scaffold (Phase 1 complete)

Planned the full project (see plan artifact from the planning session — not checked in, ask
Claude to re-derive it from this log and CLAUDE.md if it's needed again). Key decisions:

- Topology: 2 spine + 4 leaf, two-stage Clos, per site. leaf1+leaf2 = MLAG/vPC pair,
  leaf3/leaf4 standalone VTEPs.
- Two sites: **DC1 = Arista** (deployed live via containerlab), **DC2 = Cisco** (rendered only,
  via Nexus as Code's `validate` role — no Nexus Dashboard needed or available).
- Protocols: single ASN per site, OSPF area 0 underlay, iBGP EVPN overlay, spines as route
  reflectors. Chosen specifically so NetBox needs no per-device ASN field.
- NetBox model: native objects first. Config contexts (not custom object types) carry
  fabric-wide settings. Exactly 3 custom fields: `node_id`, `mlag_domain_id`, `vni`. Zero
  plugins. Target headline: 2 sites, 12 devices, 3 custom fields, 0 plugins.
- Repo layout: three separate playbook directories — `nb-playbooks/` (all NetBox read/write),
  `avd-playbooks/` (pure AVD renderer), `nac-playbooks/` (pure Nexus as Code renderer). They
  share nothing directly; `nb-playbooks/export-model.yml` writes a vendor-neutral
  `build/fabric-model.yml` plus per-site `build/dc*-inventory.yml` files that the two renderers
  read. Neither renderer talks to NetBox directly.

### Done
- `.venv` created and verified: `ansible-core` 2.21.4, `ansible-lint` 26.8.0, `yamllint` 1.38.0,
  `pyavd` 6.3.0.
- Collections installed globally: `arista.avd` 6.3.0, `netbox.netbox` 3.23.0 (already present),
  `cisco.nac_dc_vxlan` 0.5.0, `cisco.dcnm` 3.13.0 (newly installed).
- Verified the installed `cisco.nac_dc_vxlan` 0.5.0 source directly (not just docs): confirmed
  `vxlan.global.ibgp.bgp_asn` is the correct current path (with a legacy `vxlan.global.bgp_asn`
  fallback), so the planned schema is safe to build against.
- `.yamllint`, `.ansible-lint` written and passing (clean run over the whole repo).
- `requirements.txt` / `requirements.yml` pinned.
- Directory scaffolding + per-directory `ansible.cfg` for `nb-playbooks/`, `avd-playbooks/`,
  `nac-playbooks/`, plus `build/`, `docs/`, `clab/`.
- `.gitignore` extended for `build/`, `.vault_pass`, clab runtime state, AVD/NaC generated output.

### Next up — Phase 2: NetBox model loader (DC1 only)
Build `nb-playbooks/load-model.yml` + one role per object type + numbered host_vars files under
`nb-playbooks/inventory/host_vars/netbox/` (site, racks, manufacturer, platform, device type +
interface templates, device roles, devices, ASN, VLAN groups/VLANs, VRF, prefix pools/roles,
service prefixes, cables, config contexts, the 3 custom field definitions). DC1 (Arista) only —
DC2 comes later in Phase 5. Must be idempotent: running it twice should report zero changes the
second time.

Concrete first steps when resuming:
1. Define the 3 custom fields (`node_id`, `mlag_domain_id` on `dcim.device`; `vni` on
   `ipam.vlan` + `ipam.vrf`) — these have to exist before objects that use them are created.
2. Site `dc1`, rack, manufacturer `Arista`, platform `eos`.
3. Device type `cEOS-lab` with interface templates (`Management0`, `Ethernet1`-`8`,
   `Loopback0`/`1`) — do this once so device creation auto-instantiates interfaces.
4. Device roles `spine`, `leaf`, `border`, `edge-router` (last two unused for now).
5. The 6 DC1 devices, `ipam.ASN` 65001, VLAN groups + VLANs + VRF `TENANT1`, the 6 prefix pools,
   the 2 service prefixes, 10 cables, the 2 config contexts (`fabric-common`, `dc1-fabric`).
6. Assert idempotency by running the playbook twice.

Full design detail (object list, field-by-field AVD/NaC mapping table, why each custom field
exists, why `VirtualChassis` was rejected for MLAG pairing, risks list) lives in the approved
plan. If it's not available, ask Claude to reconstruct it — the decisions above plus this repo's
`CLAUDE.md` are enough to regenerate it faithfully.
