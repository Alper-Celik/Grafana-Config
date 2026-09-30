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
