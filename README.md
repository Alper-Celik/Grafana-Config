# Grafana-Config

Grafana configuration that lives outside the Nix tree: dashboards exported from
the Grafana UI (root `*.json`) and the alert rules the fleet loads
(`alerting/rules.yaml`).

## alerting/rules.yaml

Grafana file-provisioned alert rules:

| uid | fires on | for | severity |
|---|---|---|---|
| `fleet-systemd-unit-failed` | `node_systemd_unit_state{state="failed"} == 1` | 5m | warning |
| `fleet-systemd-critical-unit-down` | `max_over_time(node_systemd_unit_state{state="active", name=~"sshd\|tailscaled\|…"}[10m]) == 0` | 5m | critical |
| `fleet-scrape-target-down` | `up == 0` (any scrape target) | 10m | critical |
| `fleet-host-metrics-missing` | `up{job="integrations/unix"} offset 20m unless up{job="integrations/unix"}` | 5m | critical |

All four sit in group `fleet-availability` (folder `Fleet Alerts`, `interval:
60s`) and inherit the existing notification policy (Telegram). `noDataState` is
`OK` on every rule: an empty result is the healthy case here ("nothing failed",
"no unreachable target"), so only a real value of 1 fires.

`Alper-Celik/MyServers` consumes this file as a pinned flake input
(`hetzner/server-1/observebality-server.nix`):

```nix
services.grafana.provision.alerting.rules.path = "${inputs.grafana-config}/alerting/rules.yaml";
```

Consequences:

- The rules are **read-only in the Grafana UI** — this file is the source of
  truth. To change a rule: edit it here, bump the pin in MyServers
  (`nix flake update grafana-config`) and redeploy.
- The format is Grafana's provisioning YAML, i.e. exactly what ends up as
  `provisioning/alerting/rules.yaml` in the Grafana state directory. Each rule
  needs the `A` (instant PromQL, `datasourceUid: mimir`) → `B` (threshold
  expression, `condition: B`) query pair; `pkgs.formats.yaml`'s output is
  accepted as-is, so a rule set generated from Nix can be pasted in verbatim.
- Keep `uid`s stable. Grafana keys rule state (silences, active alerts) off the
  uid, so renaming one resets it. Retiring a rule: move its uid to `deleteRules`
  once, then drop both.

## Why Git Sync does not carry the alert rules

The Grafana instance (`observe.lab.alper-celik.dev`, v13.0.9) has a Git Sync
connection on this repository (Administration → General → Provisioning →
`Alper-Celik/Grafana-Config`, branch `main`, folder sync, `write` + `branch`
workflows) and that is how the dashboards get in. Alert rules cannot ride it:

- Git Sync syncs **dashboards and folders only**: "Alerts, data sources, panels
  and other resources are not supported yet" (Git Sync — usage and performance
  limitations, Grafana v13 docs). Teaching the Git Sync engine to read alerting
  provisioning files is still open upstream (grafana/grafana#125453 epic,
  grafana/grafana#129688), and it is not behind a feature toggle — the
  `provisioning*` toggles in Grafana 13 cover folder metadata, export, readmes
  and user attribution only.
- Alerting file provisioning only accepts `.yaml`, `.yml` and `.json`
  (`pkg/services/provisioning/alerting/config_reader.go`), which is also the
  extension set the Git Sync engine treats as resources
  (`pkg/registry/apis/provisioning/resources/filepath.go`) — there is no
  extension that both Grafana's alerting provisioner and Git Sync ignore.

So the rules reach Grafana through file provisioning (path above), never through
the Git connection, and they stay read-only in the UI. One consequence of keeping
them in this repository: the connection has no `spec.github.path`, so it scans
the whole tree and `alerting/rules.yaml` looks like a resource candidate to it.
Grafana cannot parse it as a Kubernetes resource, so the pull that picks up a
change to this file reports it as an invalid resource — a per-resource
**warning** (`ReasonResourceInvalid`), not a failure, and the dashboards in the
same pull still apply. Scoping the connection to a subdirectory
(`spec.github.path`) would drop that warning, at the price of moving the
dashboards into the same subdirectory.

`README.md` is ignored by the connection for the same reason — `.md` is only
readable, never synced.
