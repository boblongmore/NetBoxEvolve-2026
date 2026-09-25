# nac-playbooks

Renders DC2's NetBox model into a Cisco Nexus as Code (NaC) `vxlan:` service model and validates
it offline with `cisco.nac_dc_vxlan.validate`. This directory needs no NetBox credentials of its
own — `nb-playbooks` owns those and is invoked automatically.

See [`docs/data-model.md`](../docs/data-model.md) for what's in NetBox and
[`docs/mapping.md`](../docs/mapping.md) for how each field becomes a `vxlan:` key.

## Prerequisites

- The Python venv at the repo root, activated: `source ../.venv/bin/activate`
- NetBox reachable and populated (`nb-playbooks/load-model.yml` already run)

No Nexus Dashboard (NDFC) instance is required — `validate` runs entirely offline.

## Run it

From inside this directory:

```bash
ansible-playbook render-nac-vars.yml   # NetBox -> host_vars/DC2/vxlan.yml
ansible-playbook build.yml             # vxlan.yml -> validate -> service model JSON
```

`render-nac-vars.yml` always reports changed — it re-exports the NetBox model on every run so
the rendered model here can never go stale silently. Re-run it any time DC2's NetBox data
changes, before re-running `build.yml`.

`build.yml` writes two files to `rendered/`:

- `dc2_service_model_golden.json` — the validated model as NaC itself understands it
- `dc2_service_model_extended.json` — the fully expanded model, the direct analogue of AVD's
  intended EOS configs on the Arista side

## Why there's no deploy.yml

A `deploy.yml` here would use `cisco.nac_dc_vxlan.dtc.create`/`dtc.deploy` to push this model to
a live Nexus Dashboard. This project doesn't have an NDFC instance to deploy against, so that
step is intentionally left out rather than built and left untested. DC2 is proven up to the
point `validate` can prove it: a real, schema-correct service model with zero validation
failures, generated from the exact same NetBox objects driving DC1.
