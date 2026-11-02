# dalinar-cos-config

[![check](https://github.com/jaborvs/dalinar-cos-config/actions/workflows/check.yml/badge.svg)](https://github.com/jaborvs/dalinar-cos-config/actions/workflows/check.yml)

Prometheus alert rules for *dalinar*, a single-server homelab, loaded into
[COS Lite](https://charmhub.io/topics/canonical-observability-stack) by the
[`cos-configuration-k8s`](https://charmhub.io/cos-configuration-k8s) charm.

## Layout

| Path | Contents |
|---|---|
| `alerts/host.yml` | The dalinar host: memory, OOM kills, disk space, load |
| `alerts/adguard.yml` | AdGuard Home, the LAN DNS, running as an LXD instance |
| `alerts/ceph.yml` | Single-node Ceph: health, OSDs, placement groups, capacity, missing metrics |
| `tests/` | `promtool` unit tests, one file per rule file |

Every alert sets `severity` to `critical` or `warning`. Alertmanager routes on that label.

## Deployment

The charm clones this repository and reads every rule file under `alerts/`:

```bash
juju config cos-config \
  git_repo=https://github.com/jaborvs/dalinar-cos-config.git \
  git_branch=main \
  prometheus_alert_rules_path=alerts
juju integrate cos-config:prometheus-config prometheus:prometheus-config
```

Whatever is on `main` goes live on the next sync, so check changes locally before pushing.
The charm also adds Juju topology labels to each rule and a prefix to each group name.
Write the rules without them.

## Checks

```bash
promtool check rules alerts/*.yml
promtool test rules tests/*.yml
```

CI runs both on every push and pull request.
To change a threshold or a `for:` duration, update the test that pins it in the same commit.

## License

[MIT](LICENSE)
