# Progress Log

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
