# stack-grafana

Monitoring stack with Grafana, for k8s.

This should really be ``stack-monitoring`` and the repo name will probably get updated at some point.

Each component is a single instance on a node labelled `grafana_core_<component>` (set in iac-homelab):

| Component | Kind | Data | Reached as |
| --- | --- | --- | --- |
| Grafana | StatefulSet | `$MONITORING_STORAGE_CLASS` volume (users, anything made in the UI) | `grafana.$DOMAIN` |
| VictoriaMetrics | StatefulSet | `$MONITORING_STORAGE_CLASS` volume, `$MONITORING_RETENTION` | `victoriametrics.$DOMAIN` |
| VictoriaLogs | StatefulSet | `$MONITORING_STORAGE_CLASS` volume, `$MONITORING_RETENTION`, syslog from rsyslog on 514 | `victorialogs.$DOMAIN` |
| vmagent | Deployment | emptyDir buffer | `vmagent.$DOMAIN` |
| vmalert | Deployment | - | `vmalert.$DOMAIN` |
| Alertmanager | Deployment | emptyDir (silences are lost on restart) | `alertmanager.$DOMAIN` |
| node_exporter | DaemonSet, every node | - | scraped by vmagent |

Longhorn (`stack-longhorn`) has to be deployed first, `deploy` stops if it isn't running. The namespace is privileged for node_exporter, which uses the host's network, PIDs and filesystem.

The components talk to each other over TLS by their service names (`<component>.grafana-core.svc`), with certs from the stack's CA (`secrets/ca.crt`, regenerated when their names change). nginx proxies `<component>.$DOMAIN` to each. Grafana's data sources go straight to the services, verified with the CA.

vmagent (`configs/vmagent`) finds node_exporter and the Vault pods through the Kubernetes API (a ClusterRole, see `k8s/rbac.yaml`) and scrapes the management VMs' node_exporter and RabbitMQ directly.

The `*.tmpl` configs are rendered with a fixed list of variables (`CONFIG_TEMPLATE_VARS` in `stack`), so Grafana's `$__file{}` is left alone. The dashboards in `configs/grafana/dashboards` are picked up by Grafana while it runs, everything else restarts the pods when it changes. Grafana installs its plugins on start (`GF_PLUGINS_PREINSTALL`).

`clean` keeps the data volumes, `purge` removes them.

## Deployments

- `core/production` - the `production.core` cluster (was `services/production` on swarm).
- `core/development` - the `development.core` cluster (to be built).

## Credits

Nginx dashboard is heavily based on: <https://grafana.com/grafana/dashboards/24774-nginx-with-vlogs>
