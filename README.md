# shard-proxy Helm chart

Helm chart for [shard-proxy](https://github.com/lightwebinc/shard-proxy) — the IPv6 multicast frame proxy in the BSV multicast transaction distribution pipeline.

This repository packages templates, default values, JSON Schema validation, and CI workflows for the proxy. The application source lives in [`shard-proxy`](https://github.com/lightwebinc/shard-proxy).

## Install

> The chart references `ghcr.io/lightwebinc/shard-proxy:<appVersion>`. Until the corresponding image is published from the application repo, `helm install` will succeed in rendering but pods will `ImagePullBackOff`.

```bash
# OCI registry
helm install proxy oci://ghcr.io/lightwebinc/charts/shard-proxy \
  --version 0.1.0 -n bsv-mcast --create-namespace \
  --set networking.multus.fabricIPv6=fd20::21/64

# Or from a local clone
helm install proxy . -n bsv-mcast --create-namespace \
  --set networking.mode=host
```

## Networking modes

| Mode | Description |
|---|---|
| `multus` (default) | Primary CNI for control/metrics; macvlan secondary `net1` on dedicated fabric NIC. Requires a `NetworkAttachmentDefinition` named `mcast-fabric` in namespace `bsv-mcast`. |
| `host` | `hostNetwork: true`; `MULTICAST_IF` resolves to the host NIC named by `config.multicastIf`. Single-NIC fallback. |
| `unicast` | Reserved for a future proxy `EGRESS_MODE=unicast-list` release. The chart renders pods but emits a `helm.sh/chart-warnings` annotation. |

See the [composition spec](https://github.com/lightwebinc/bsv-multicast/blob/main/containerization/composition-spec.md) for wiring proxy + listener + retry-endpoint via Helmfile / ArgoCD / Terraform / plain Helm.

## Values reference

See [`values.yaml`](values.yaml) for the full annotated reference. Every flag accepted by the proxy binary is exposed under `.config`; cluster-shape knobs (replicas, autoscaling, PDB, NetworkPolicy, ServiceMonitor) live at the top level.

The chart includes [`values.schema.json`](values.schema.json) — `helm install` rejects out-of-range `shardBits`, invalid `mcScope`, or invalid `networking.mode` before reaching the cluster.

### SSM (Source-Specific Multicast)

`config.sourceMode` (`asm` default, `ssm` opt-in) renders to the
`SOURCE_MODE` env var. When `ssm`, set `config.bindSource` to the
per-pod IPv6 from your Multus/Whereabouts allocation — each replica
MUST hold a distinct address (anycast/ECMP-shared sources break
PIM-SSM RPF). `bindSource` renders to `BIND_SOURCE`. See the
[SSM Support Plan](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/SourceSpecificMulticast/ssm-support-plan.md)
for fabric prerequisites.

## Helm test

```bash
helm test proxy -n bsv-mcast
```

Probes `/healthz` and `/metrics` on the metrics Service.

## Release

The `release.yml` workflow is gated. It runs only via `workflow_dispatch` with `confirm: RELEASE` and a `production` GitHub Environment review. Tag-based auto-release is intentionally disabled.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
