# NetBox Evolve 2026

## Building a Common Control Plane for Multi-Vendor Fabrics

The purpose of this project is to show building a common data model that can be used across Arista AVD and Cisco NAC for building Data Center Fabrics. The philosophy is to use NetBox native objects as the first choice and create customer fields and custom objects only where it makes sense.

`nb-playbooks` owns NetBox: it loads the fabric model in and exports a single vendor-neutral
model out. `avd-playbooks` and `nac-playbooks` are pure renderers on top of that the NetBox data model.

This repo was put together based on past experience and is presented for the NetBox Evolve 2026 Conference.

## Documentation

| Doc | What's in it |
|---|---|
| [`docs/data-model.md`](docs/data-model.md) | The NetBox objects, custom fields, and config contexts that make up the fabric model, and why each one was modeled that way |
| [`docs/mapping.md`](docs/mapping.md) | Field-by-field reference: how each NetBox object becomes an AVD `eos_designs` key and a Nexus as Code `vxlan:` key |
| [`avd-playbooks/README.md`](avd-playbooks/README.md) | How to run the Arista AVD pipeline — render, build, deploy, validate |
| [`nac-playbooks/README.md`](nac-playbooks/README.md) | How to run the Cisco Nexus as Code pipeline — render, build (offline validate) |

References:
- **Arista AVD**: https://avd.arista.com/6.4/

- **Cisco NAC**: https://netascode.cisco.com/
