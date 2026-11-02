# dalinar-cos-config

Prometheus alert rules for dalinar, my single-server homelab, running on COS Lite.

The [`cos-configuration-k8s`](https://charmhub.io/cos-configuration-k8s) charm clones this repo and loads every rule file under `alerts/` into the Prometheus of [COS Lite](https://charmhub.io/topics/canonical-observability-stack). Alertmanager routes each alert to my phone through [ntfy](https://ntfy.sh), based on its `severity` label. Whatever is on `main` goes live on the charm's next sync, so every change is checked with `promtool` first.

## What it watches

| Alert | Severity | Fires when |
|---|---|---|
| `HostMemoryPressureHigh` | warning | Less than 10% of RAM available for 10 minutes |
| `HostOOMKill` | critical | The kernel OOM-killed a process in the last 5 minutes |
| `HostDiskSpaceLow` | warning | A filesystem (not tmpfs) has less than 15% free for 10 minutes |
| `HostHighLoad1` | warning | load1 stays above 1.5x the CPU count for 10 minutes |
| `AdGuardOOMKilled` | critical | The AdGuard Home LXD instance was OOM-killed in the last 5 minutes |
| `CephHealthError` | critical | Ceph reports `HEALTH_ERR` |
| `CephOSDDown` | critical | An OSD has been down for 5 minutes |
| `CephPGsNotClean` | critical | Any placement group is degraded, undersized, incomplete, down, inconsistent or stale for 10 minutes |
| `CephPoolNearFull` | warning | More than 85% of the cluster's raw capacity is used |
| `CephMetricsAbsent` | warning | No Ceph metrics have arrived for 10 minutes |

- `HEALTH_WARN` doesn't alert. Ceph runs on a single node with size-1 pools, so it sits in `WARN` all the time.
- There's no swap rule. Neither machine has swap, so available memory is the early warning and an OOM kill is the hard failure.
- `summary` and `description` are the text of the phone notification, so they're short and include the value that fired.
- The rules carry no Juju topology labels. The charm adds them and prefixes each group name, so `dalinar-host` shows up in Prometheus as `dalinar_host_rules`.

## Requirements

- COS Lite deployed with Juju, with Prometheus and Alertmanager.
- The metrics the rules read: `node_*` from node_exporter, `lxd_*` from the LXD metrics endpoint and `ceph_*` from the Ceph manager's Prometheus module. Here Grafana Alloy collects all three and pushes them to Prometheus.
- `promtool` from a [Prometheus release](https://github.com/prometheus/prometheus/releases), to check changes. CI uses 3.15.1.

## Install

```bash
juju deploy cos-configuration-k8s cos-config
juju config cos-config \
  git_repo=https://github.com/jaborvs/dalinar-cos-config.git \
  git_branch=main \
  prometheus_alert_rules_path=alerts
juju integrate cos-config:prometheus-config prometheus:prometheus-config
```

Name the relation in full. `cos-config` and `prometheus` can relate in two ways, so `juju integrate cos-config prometheus` on its own is ambiguous.

To check that the rules loaded, open Prometheus at *Alerts*. The three groups should show up with the ten rules in `inactive`.

## Layout

| Path | What's in it |
|---|---|
| `alerts/host.yml` | The dalinar host: memory, OOM kills, disk space and load |
| `alerts/adguard.yml` | AdGuard Home, the LAN DNS |
| `alerts/ceph.yml` | Ceph: health, OSDs, placement groups, capacity and missing metrics |
| `tests/` | `promtool` unit tests, one file for each rule file |

## Changing a rule

Edit the rule and the test that covers it in the same commit, then run:

```bash
promtool check rules alerts/*.yml
promtool test rules tests/*.yml
```

CI runs both on every push and pull request. It downloads `promtool` and checks the tarball's SHA-256 against the release before running it.

## Tests

Each test feeds synthetic series to one rule and checks which alerts are firing at given times, with their labels and exact notification text. Most also check the edge, for example memory at exactly 10% stays quiet, and an OSD down for 4 minutes doesn't fire but at 5 it does. Others check the cases that must stay quiet: tmpfs mounts, Ceph in `HEALTH_WARN` and OOM kills in other instances. They don't check that the metrics exist in a live Prometheus.

## License

MIT, see [LICENSE](LICENSE).
