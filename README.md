# OKP Helm Chart

Helm chart for deploying [Red Hat Offline Knowledge Portal (OKP)](https://docs.redhat.com/en/documentation/red_hat_offline_knowledge_portal) on OpenShift.

OKP provides offline access to Red Hat product documentation through a bundled Apache httpd server (serving HTML docs) and Apache Solr (search and RAG index). It is designed to work with [Intelligent Assistant for Developer Hub](https://docs.redhat.com/en/documentation/red_hat_developer_hub) to deliver grounded, citation-backed AI responses from product documentation — including in air-gapped environments with no internet access.

## Prerequisites

- OpenShift cluster (for Route support)
- Access to `registry.redhat.io` (or a mirrored copy of the OKP image)

## Quick Start

```bash
helm install okp ./charts/okp -n <namespace>
```

## Uninstall

```bash
helm uninstall okp -n <namespace>
```

## Configuration

All configuration is in [`charts/okp/values.yaml`](charts/okp/values.yaml).

| Parameter | Description | Default |
|---|---|---|
| `image.repository` | OKP container image | `registry.redhat.io/offline-knowledge-portal/rhokp-rhel9` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `replicas` | Number of pod replicas | `1` |
| `solr.memory` | JVM heap size for Solr | `1g` |
| `solr.hostBind` | Solr bind address | `0.0.0.0` |
| `httpd.serverName` | Apache ServerName directive | `localhost` |
| `httpd.compressed` | Whether docs on disk are gzip-compressed | `true` |
| `httpd.encrypt` | Whether docs on disk are encrypted | `false` |
| `resources.requests.cpu` | CPU request | `200m` |
| `resources.requests.memory` | Memory request | `2Gi` |
| `resources.limits.cpu` | CPU limit | `2` |
| `resources.limits.memory` | Memory limit | `4Gi` |
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `route.enabled` | Create an OpenShift Route | `true` |
| `route.tls.termination` | TLS termination type | `edge` |
| `route.tls.insecureEdgeTerminationPolicy` | Insecure traffic policy | `Allow` |

## Architecture

The OKP container bundles two services managed by the chart:

- **Apache httpd** (port 8080) — Serves HTML documentation from `/var/www/html/` and proxies `/solr` requests to the Solr backend. This is the primary entry point exposed via the Service and Route.
- **Apache Solr** (port 8983) — Provides the search index used by Lightspeed Core Service (LCORE) for Retrieval Augmented Generation (RAG) queries.

The documentation files on disk are gzip-compressed by default (`httpd.compressed: "true"`), and httpd applies a `gunzip` output filter before serving them to browsers.

## Integration with Lightspeed Core Service

To connect LCORE to this OKP instance, configure the OKP section in your `lightspeed-stack.yaml`:

```yaml
rag:
  inline:
    - okp
okp:
  rhokp_url: http://<okp-service-name>.<namespace>.svc.cluster.local:8080
  offline: true
  chunk_filter_query: "product:*developer_hub*"
```

For external access (e.g., citation links opening in a browser), use the Route URL instead:

```yaml
okp:
  rhokp_url: https://<okp-route-hostname>
```

## Using as a Subchart

This chart can be used as a dependency in another Helm chart:

```yaml
# In the parent chart's Chart.yaml
dependencies:
  - name: okp
    version: "0.1.0"
    repository: "file://path/to/rhdh-okp-chart/charts/okp"
    condition: okp.enabled
```

Then pass values under the `okp:` key in the parent chart's `values.yaml`.
