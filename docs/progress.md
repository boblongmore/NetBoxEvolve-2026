# Progress Log

## 2026-09-25 (later) — Each renderer's pipeline is now self-sufficient

Operational feedback: an operator working in `avd-playbooks` or `nac-playbooks` shouldn't have to
manually `cd ../nb-playbooks` first to get fresh NetBox data. Rather than duplicate
`fabric_export`'s NetBox-reading logic (or its credentials) into each renderer directory --
which was explicitly rejected back in the Phase 2 design discussion, for the same reasons --
`render-avd-vars.yml` and `render-nac-vars.yml` now each start with a task that shells out to
`nb-playbooks/export-model.yml` (`ansible.builtin.command` with `chdir`). `nb-playbooks` stays
the sole owner of NetBox credentials and read logic; each renderer's entry point just triggers
it as its own first step. Verified both run correctly with no prior manual step, from a shell
that's never `cd`'d into `nb-playbooks` at all.

Deliberately left `render-clab.yml` unchanged -- generating the containerlab topology is a
one-time lab-setup step, not part of the recurring build/deploy/validate cycle this addresses.

## 2026-09-25 — DC2 (Cisco) modeled, exported, and validated (Phase 5 complete)

Extended NetBox with DC2 (site, rack, `Cisco` manufacturer, new `nxos` platform, new
`N9K-C93180YC-EX` device type with real NX-OS interface naming (`Ethernet1/N`, `mgmt0`), 6
devices, cables, ASN 65002, its own prefix pools out of `10.2.0.0/16`, its own VLAN
groups/VLANs, and a `dc2-fabric` config context) -- shared objects (device roles, VRF, tenant)
reused as-is. Then built `nac-playbooks` end to end: exported `build/dc2-model.yml`, rendered it
into Nexus as Code's `vxlan:` schema, and ran `cisco.nac_dc_vxlan.validate` for real --
**0 failures**, golden and extended service model JSON written to `nac-playbooks/rendered/`
(the extended one is the direct analogue of AVD's structured configs).

**The big payoff from Phase 3's design:** `nb-playbooks/roles/fabric_export` was built
site-parameterized from the start (`fabric_export_site` role default). Extending it to DC2
needed zero code changes to the role itself -- just a second `include_role` call with
`fabric_export_site: dc2` overridden. The "one model, two vendors" thesis held up exactly as
intended.

**Real design fix made before extending, not after:** the VRF's RD was hardcoded as
`"65001:50001"` (DC1's ASN) in Phase 2. Copying that straight to DC2 (ASN 65002) would have been
silently wrong -- a real error a network engineer in the audience would catch. Removed the
stored RD entirely (both AVD and Nexus as Code derive it correctly from each site's own ASN by
default); this needed setting NetBox's field to explicit `rd: null`, not `""` -- NetBox
normalizes an empty string back to null on read, so `""` looked "changed" on every single run
forever until caught.

**Real bugs hit and fixed extending nb-playbooks to a second site (all found by actually running
it, not by inspection):**
1. `devices: "{{ devices | selectattr(...) }}"` in an `include_role` `vars:` block recurses
   infinitely -- Ansible resolves the RHS against that same `vars:` block once a key of that
   name is being redefined there, not the outer host var. Fixed by referencing
   `hostvars[inventory_hostname].devices` explicitly instead of the bare name.
2. `fabric_export`'s several accumulator facts (`fabric_export_devices_raw`, `_cables`, `_pools`,
   etc.) are built with `default([]) + [...]`, and `set_fact` persists as a host fact for the
   rest of the play -- running the role a second time (for DC2) would have silently appended
   DC2's data onto DC1's leftovers instead of starting clean. Added an explicit reset task at
   the top of the role.
3. The new "scope the VTEP VIP lookup to this site's vtep-loopback pool" code (written today,
   to fix the single-MLAG-pair-globally assumption once a second pair existed) crashed on
   `selectattr('role.name', ...)` against service prefixes, which have `role: null` --
   `None.name` has no attribute. Fixed with `rejectattr('role', 'none')` first.
4. The single global MLAG/vPC VTEP VIP (`role=anycast`, no site filter) from Phase 3 needed to
   become per-site once DC2 added a second one -- scoped by "falls inside this site's own
   vtep-loopback pool" (`?parent=<pool>`) rather than adding a new NetBox field just to
   disambiguate.
5. Two site-dependent things the `devices` role hardcoded turned out not to be constants:
   the management interface name (`Management0` for cEOS, `mgmt0` for N9K) and the device
   type/platform/rack. All now role defaults, overridden per site at `include_role` time.

**Real bugs hit and fixed building the Nexus as Code renderer (none of this existed before
today, so none of it is a repeat of an earlier mistake):**
1. `cisco.nac_dc_vxlan.validate`'s own data-file scanning is hardcoded to
   `playbook_dir/host_vars/<inventory_hostname>/*.yml` -- a completely separate mechanism from
   Ansible's own inventory-relative `host_vars/` (which is where `ansible_network_os`, needed
   for a different reason below, correctly lives). Confirmed directly in the role's
   `sub_main.yml`. Two parallel `host_vars/` trees, different consumers, easy to conflate.
2. The role's own workflow-detection debug task dereferences `ansible_network_os` unconditionally
   and errors if it's undefined. Not something NetBox generates -- added
   `nac-playbooks/inventory/host_vars/DC2/connection.yml` (hand-authored, `cisco.nxos.nxos`,
   accurate for what we're actually modeling).
3. Missing Python deps the collection's own `requirements.txt` names explicitly:
   `nac-yaml==1.0.0` and `nac-validate==1.0.0` -- pinned exactly, not "latest", since 2.0.0 of
   each has a different internal module layout the collection doesn't import correctly against.
4. A real bug in the collection's own code: `when: enhanced_rules_path is defined and
   enhanced_rules_path` returns a bare (falsy) string when that optional var is set to `""`, and
   this ansible-core version's stricter conditional type-checking rejects a non-boolean `when`
   result outright. We don't use the enhanced-rules feature at all. Set
   `allow_broken_conditionals = True` in `nac-playbooks/ansible.cfg` -- Ansible's own documented
   escape hatch for exactly this kind of old-collection/new-engine mismatch, named directly in
   the error message.
5. `validate` in this installed version (0.5.0) only keeps the golden/extended service model as
   in-memory facts (`MD`/`MD_Extended`) -- it doesn't write them to a file itself (that's
   apparently newer behavior; my original research was against 0.9.0-dev source, which does).
   Added two small tasks to `build.yml` to persist them to `nac-playbooks/rendered/*.json`.

### What's committed vs. not
Same distinction as Phase 4: `build/*.yml` stays gitignored (interchange only).
`nac-playbooks/host_vars/DC2/vxlan.yml` and `nac-playbooks/inventory/host_vars/DC2/connection.yml`
ARE committed -- they're the renderer's actual input, same treatment as
`avd-playbooks/inventory/group_vars/*`. `nac-playbooks/rendered/*.json` (the golden/extended
service model) stays gitignored -- fully-derived final output, same as `avd-playbooks/intended/`.

### Next up
The core "one model, two vendors" thesis is now fully proven end to end: DC1 deployed live via
AVD with a healthy EVPN control plane, DC2 rendered and validated via Nexus as Code with zero
errors, both generated from the same NetBox objects. Remaining optional work: the border-leaf
phase (additive, `border`/`edge-router` roles already exist unassigned), a `docs/mapping.md`
field-by-field writeup now that both renderers are real, and deciding whether the untested
"`dtc.common` might render full NDFC payloads offline" idea from the original plan is worth the
~30 minutes on the bench.

## 2026-09-24 — Deployed and validated live (Phase 4 truly complete)

Built `avd-playbooks/deploy.yml` (`arista.avd.eos_config_deploy_eapi`) and `validate.yml`
(`arista.avd.anta_runner`) and ran both against the live lab. Result: **BGP EVPN established
to both spines, MLAG up, all 4 VTEPs discovered, VXLAN/OSPF/MLAG/Routing/Connectivity/STP/
Configuration ANTA categories 100% passing** (128/146 tests; the only 18 failures are
Management0 discard counters, benign cEOS hardware-ID/ACL log noise, and NTP unsynchronized --
all expected artifacts of an isolated container lab with no real NTP server, none touching
fabric correctness).

Getting there took real, live troubleshooting -- worth recording in detail since the model and
playbooks were all correct going in; every issue below was a genuine EOS/AVD/lab behavior, not
a repeat of an earlier mistake:

1. **`configure replace` is just slow on this cEOS image.** The `eos_config_deploy_eapi` role's
   default 30s command timeout was too short, and the failure mode was actively misleading:
   some devices reported "Invalid input" (looked like a syntax error), others a plain timeout --
   both were actually the same thing, confirmed by manually replaying the operation outside
   Ansible with a long timeout and watching it succeed server-side well past 30s. Fixed by
   setting `ansible_command_timeout: 240` in `group_vars/all/connection.yml` -- ended up needing
   more than one bump (30s, then 120s still wasn't enough for the MLAG pair's larger config,
   then 240s).
2. **A genuine invalid-VLAN bug, found underneath the timeout noise.** Once timeout stopped
   being the variable, `dc1-leaf1`/`dc1-leaf2` (the MLAG pair) kept failing with a real,
   immediate rejection: `vlan 53000` -- above EOS's 4094 maximum. Root cause: AVD's optional
   "MLAG iBGP peering per VRF" feature auto-derives that VLAN ID from `vrf_vni` when `vrf_id`
   isn't set, and our VRF's 5-digit VNI (50001) pushed the formula miles out of range. Fixed by
   setting `enable_mlag_ibgp_peering_vrfs: false` on the tenant in `fabric.yml.j2` -- simpler
   than plumbing a second small integer identifier into the model just to satisfy this one
   formula, and the feature is optional extra resiliency, not required for fabric correctness.
3. **Our generated config had no way to keep itself reachable after a real replace.** Confirmed
   the hard way: a `configure replace` had already succeeded on `dc1-spine1`, `dc1-spine2`,
   `dc1-leaf3`, `dc1-leaf4` in the very first deploy attempt (the "invalid input"/timeout errors
   masked this), but since our AVD input never set `management_eapi` or `aaa_settings.local_users`,
   the replace correctly wiped eAPI and the admin account right along with everything else,
   locking every one of those devices out of further Ansible management. Recovered via
   `docker exec` console access (bypasses the network) on each affected device, then fixed
   properly by adding `management_eapi.enabled: true` and an `aaa_settings.local_users` entry
   (reusing the same vaulted `ansible_user`/`ansible_password`, via `hostvars[groups['DC1'][0]]`,
   so the fabric keeps using the identical admin account rather than a second one) to
   `fabric.yml.j2`.
4. **Also self-inflicted, also recovered the same way:** manually testing `configure replace`
   with a deliberately minimal candidate config (to reproduce the syntax error safely) removed
   `dc1-leaf1`'s management IP and eAPI config for real, since a genuine `configure replace`
   removes anything not present in the candidate. Recovered via console; a useful reminder that
   testing `configure replace` against a live device is not actually a safe no-op operation.

None of this required changing the underlying data model -- every fix lived in `fabric.yml.j2`
or `connection.yml`. `nb-playbooks` and `build/dc1-model.yml` were untouched this session.

### Next up
Phase 5: DC2 (Cisco) objects in NetBox + the Nexus as Code renderer (`nac-playbooks/`),
rendered only via `cisco.nac_dc_vxlan.validate` -- no live deployment, no Nexus Dashboard.
Remember the `nac-playbooks` ansible.cfg will need the same inventory-symlink fix already
applied to `avd-playbooks` (point at `inventory/` with `inventory/hosts.yml` symlinked to
`../build/dc2-inventory.yml`) so its own group_vars get discovered correctly from the start.

## 2026-09-21 (latest) — AVD renderer for DC1, live (Phase 4 complete)

Built the full Arista renderer and ran it end-to-end against the containerlab lab the user
deployed from `clab/dc1-arista.clab.yml` (6 cEOS containers, `admin`/`admin` eAPI creds --
already valid out of the box on this image, vaulted anyway for consistency). Confirmed
mgmt IPs matched NetBox exactly and eAPI (443) was reachable before building anything.

**New pieces:**
- `clab/dc1-arista.clab.yml` -- generated by `avd-playbooks/render-clab.yml`
  (`roles/clab/`) straight from `build/dc1-model.yml`. Validated with `containerlab graph`
  before the user deployed it. Mgmt network is `clab-dc1-mgmt` on `172.20.21.0/24`, distinct
  from the sibling `avd-poc` project's live `clab-mgmt` on `172.20.20.0/24`.
- `avd-playbooks/render-avd-vars.yml` (`roles/avd_inputs/`) -- templates `eos_designs` input
  from `build/dc1-model.yml` into `group_vars/all/fabric.yml` (protocols, tenants/VRFs/VLANs),
  `group_vars/dc1_spine/spine.yml`, `group_vars/dc1_leaf/leaf.yml` (MLAG node_group + per-node
  uplinks, all derived from `dc1_model.devices[].uplinks`, never hand-listed).
- `avd-playbooks/build.yml` -- runs `arista.avd.eos_designs` then `arista.avd.eos_cli_config_gen`.
  Verified output: correct OSPF underlay on the right interfaces, MLAG domain 1 /
  Port-Channel1, iBGP EVPN with spines as `route-reflector-client` peers to all 4 leaf
  loopbacks, VRF TENANT1 (VNI 50001, RD 65001:50001), VLANs 110/120 with correct VNIs.

**Research done properly, not guessed:** delegated a focused read of the *installed*
`arista.avd` 6.3.0 collection's own docs/examples (not general knowledge) for the exact
`eos_designs` schema -- confirmed `spine`/`l3leaf` are stock `node_type_keys` (no custom ones
needed), the 5 IP pool key names, that `node_groups` with 2 nodes auto-enables MLAG, and that
VLAN/VRF `filter` defaults to `['all']` so every l3leaf gets every tenant service with no
`tags:` filter needed (matches this fabric, where all 4 leafs carry `svc-tenant1`).

**Real bugs hit and fixed, in order:**
1. Missing Python deps (`distlib`, `deepmerge`, `aristaproto`, `grpclib`, `pyavd-utils`,
   `python-socks`) -- same ones the sibling `capitalone-avd-poc` project already pins. Added
   to `requirements.txt`.
2. AVD requires `fabric_name` to be the literal name of an Ansible inventory group containing
   every fabric host, not just a string label. Fixed at the source: rewrote
   `nb-playbooks/roles/fabric_export/templates/inventory.yml.j2` to nest the role groups
   (`dc1_spine`, `dc1_leaf`) under a parent group named after `fabric.name` (`DC1`).
3. `hosts: "{{ fabric_name }}"` doesn't work -- group_vars aren't available for templating the
   `hosts:` keyword itself at parse time. `build.yml` uses the literal group name `DC1`
   instead, which is fine since this playbook is already DC1-specific by construction.
4. The real bug behind bug 2 looking like it "should have" already worked: `avd-playbooks`'s
   `ansible.cfg` pointed `inventory` directly at `../build/dc1-inventory.yml`, so Ansible's
   group_vars auto-discovery looked for `group_vars/` next to `build/`, not next to
   `avd-playbooks/inventory/` where the AVD input files actually live. Fixed by pointing
   `inventory` at the `inventory/` directory instead, with `inventory/hosts.yml` as a symlink
   to `../../build/dc1-inventory.yml` -- keeps `build/` the single source of truth while
   satisfying Ansible's adjacency requirement. Same fix will be needed for `nac-playbooks` later.
5. This cEOS image names its OOB port `Management0`; AVD's default is `Management1` and the
   `cEOSLab` platform_settings entry doesn't override it. Read pyavd's actual installed source
   (`_eos_designs/shared_utils/mgmt.py`) to get the real resolution order rather than guessing
   further from docs prose: per-node `mgmt_interface` (via type `defaults`) beats
   `platform_settings`, which beats the fabric-wide `mgmt_interface_settings`/`mgmt_interface`
   vars. Set `mgmt_interface: Management0` under `spine.defaults` and `l3leaf.defaults` --
   the fabric-wide vars alone never took effect despite matching the documented key names.
6. `ansible-lint --fix=jinja` (used on the `fabric_export` role in Phase 3) produces valid but
   much less readable output -- collapses multi-line dict literals into single quoted strings
   full of escaped `\n`. Reverted; moved dict construction into task-level `vars:` blocks
   (native YAML, each field its own templated value) instead, which sidesteps the lint rule
   entirely rather than fighting its formatter.
7. `exclude_paths` in `.ansible-lint` resolves relative to wherever `ansible-lint` is invoked
   from, not the config file's own location -- needed both `intended/` and
   `avd-playbooks/intended/` forms since commands run from inside each playbook directory.

**What's committed vs. not:** `build/*.yml` stays gitignored (pure interchange, regenerated
every run). `avd-playbooks/inventory/group_vars/*` and `clab/dc1-arista.clab.yml` ARE
committed despite being generated -- unlike `build/`, they're meaningful, reviewable
configuration (the actual AVD input, the actual deployment manifest), not disposable
plumbing. `avd-playbooks/intended/` (structured configs, final `.cfg` files) and
`avd-playbooks/documentation/` stay gitignored -- fully-derived final output, already covered
by `.gitignore` since Phase 1.

Also published two topology diagrams outside this repo: a Claude Artifact
(https://claude.ai/artifact/1jxLbPhjKs3uhETyMBztkx) and a Lucid document named
"Netbox-evolve-avd" (https://lucid.app/lucidchart/b75d99fb-792e-476f-bd7b-1b917b8f155f/edit).
Neither is tracked in git; both were built from the same cable-graph data as everything else.

### Next up
Not yet deployed live -- `build.yml` only generates local structured configs and intended
`.cfg` files, it doesn't touch the running containers. Next: `avd-playbooks/deploy.yml`
(`eos_config_deploy_eapi`) to actually push config to the 6 running cEOS containers, then
`avd-playbooks/validate.yml` (`anta_runner`) to confirm OSPF neighbors, BGP EVPN sessions, and
VTEP discovery. After that, Phase 5: DC2 (Cisco) objects in NetBox + the Nexus as Code renderer.

## 2026-09-21 (later) — Neutral export for DC1 (Phase 3 complete)

Built `nb-playbooks/export-model.yml` (a single role, `fabric_export`). It reads DC1 back out of
NetBox and writes `build/dc1-model.yml` (the vendor-neutral fabric model) and
`build/dc1-inventory.yml` (a plain YAML Ansible inventory grouped by role, `ansible_host` = bare
mgmt IP). Neither file is committed (`build/` is gitignored) — both are regenerated by re-running
this playbook, and it's meant to be step one of the demo runbook whenever NetBox changes.

**Deviation from the original plan wording:** the plan said a single `build/fabric-model.yml`.
Built per-site instead (`dc1-model.yml`, and `dc2-model.yml` later) so each renderer
(`avd-playbooks`, `nac-playbooks`) reads its own site's file directly with no filtering logic
needed. Simpler given the renderers are already split one-per-vendor-per-site.

**Design choices:**
- Pulled everything via raw `ansible.builtin.uri` GET calls against the NetBox REST API, not
  `nb_inventory`. This was a deliberate change from the plan's original wording (which assumed
  `nb_inventory` the way `capitalone-avd-poc` uses it) — `nb_inventory` is device-centric and has
  no facility for VLANs/VRFs/prefixes/cables, and using two different data-access mechanisms in
  the same role added more complexity than it saved. It also sidesteps `nb_inventory`'s documented
  wart of stripping `primary_ip4` to a bare address (irrelevant now — the raw API always returns
  the full CIDR).
- Used `?include=config_context` on the devices query to get NetBox's own already-merged
  config-context data, rather than reimplementing the weight-based layering logic.
- Derived `uplinks` per device and `mlag_pairs` once, centrally, in the export — not left for each
  renderer to re-derive from the raw cable list independently. Built a temporary bidirectional
  link list (one entry per direction per cable) as scratch data to make per-device filtering a
  single `selectattr` instead of walking both `a`/`b` sides of every cable by hand.
- Anycast gateway (`.1` convention) computed with `ansible.utils.ipaddr('1')` — added `netaddr`
  to `requirements.txt` since the filter needs it. Preferred this over hand-rolling octet
  arithmetic in Jinja.
- The MLAG/vPC VTEP VIP is matched by "the only Anycast-role IP address in NetBox" — a documented,
  asserted (fail-loud, not silent) limitation. A second MLAG pair would need a real structured
  link between the IP and its `mlag_domain_id` instead of this cardinality shortcut.

**Bugs hit and fixed along the way:**
- The read-side REST API field for a device's role is `role`, not `device_role` — the latter is
  only the *write-side* Ansible module parameter name (`netbox_device`'s `data.device_role`).
  Cost one failed run against live NetBox before catching it.
- Forgot `delegate_to: localhost` on the one `ansible.builtin.setup` task (added only to grab
  `ansible_date_time` for the export banner) — it tried to actually SSH to the `netbox` inventory
  host and failed. Every other task in the role already had it; this one was just missed.
- `ansible-lint --fix=jinja` "fixed" the `jinja[spacing]` warnings by collapsing the multi-line
  dict literals into single quoted strings full of escaped `\n` — technically satisfies the rule,
  but is much less readable. Reverted and instead moved every constructed dict into a task-level
  `vars:` block (native YAML, each field its own templated value) rather than one large Jinja
  expression — cleaner than either the manual multi-line Jinja or the auto-fixer's rewrite, and
  it happens to sidestep the lint rule entirely rather than fighting it.
- Also caught along the way (Phase 2 gap, fixed before starting Phase 3): the VRF/VLANs were
  tagged `svc-tenant1` but the devices that should carry them never were — see the entry below.
  Also renamed the top-level `tags` variable to `netbox_tags` (collided with Ansible's reserved
  tag-filtering keyword).

### Next up — Phase 4: AVD renderer
Build `avd-playbooks/roles/avd_inputs/` to template `build/dc1-model.yml` into `eos_designs`
input, then `arista.avd.eos_designs` → `eos_cli_config_gen`, then deploy to the cEOS containerlab
topology (`clab/dc1-arista.clab.yml` — not yet written) via `eos_config_deploy_eapi`, then
validate with `anta_runner`. The clab file needs mgmt pinned to `172.20.21.0/24` to match NetBox.

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
