# avd-playbooks

Renders DC1's NetBox model into an Arista AVD fabric, builds the intended EOS configuration, and
(optionally) deploys and validates it against a live lab. This directory needs no NetBox
credentials of its own — `nb-playbooks` owns those and is invoked automatically.

See [`docs/data-model.md`](../docs/data-model.md) for what's in NetBox and
[`docs/mapping.md`](../docs/mapping.md) for how each field becomes an `eos_designs` key.

## Prerequisites

- The Python venv at the repo root, activated: `source ../.venv/bin/activate`
- NetBox reachable and populated (`nb-playbooks/load-model.yml` already run)
- For deploy/validate: a running DC1 lab reachable over eAPI (see `render-clab.yml` below)

## Run it

From inside this directory:

```bash
ansible-playbook render-avd-vars.yml   # NetBox -> group_vars (spine.yml, leaf.yml, fabric.yml)
ansible-playbook build.yml             # group_vars -> eos_designs -> intended EOS configs
ansible-playbook deploy.yml            # push intended configs to the live lab via eAPI
ansible-playbook validate.yml          # run ANTA against the live devices
```

`render-avd-vars.yml` always reports changed — it re-exports the NetBox model on every run so
the group_vars here can never go stale silently. Re-run it any time DC1's NetBox data changes,
before re-running `build.yml`.

`deploy.yml` and `validate.yml` need real devices to talk to. If you only want to see the
rendered configuration (e.g. to compare against Nexus as Code's output), `build.yml` is as far
as you need to go — the intended configs land under `intended/configs/*.cfg`.

## Optional: standing up the lab

`render-clab.yml` generates the containerlab topology file for DC1 from the same NetBox-exported
model (so its mgmt IPs and cabling always match what the fabric expects):

```bash
ansible-playbook render-clab.yml
containerlab deploy -t ../clab/dc1-arista.clab.yml
```

This is a one-time step before `deploy.yml`/`validate.yml`, not part of the regular
render → build cycle.
