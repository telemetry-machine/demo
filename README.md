# Telemetry Machine demo

The [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) on Kubernetes,
deployed with [Flux](https://fluxcd.io/), sending its telemetry to
[Telemetry Machine](https://telemetrymachine.com). It's the data behind
[telemetrymachine.com/demo](https://telemetrymachine.com/demo).

It also shows how to connect a Kubernetes cluster to Telemetry Machine: the
demo's services, and the cluster itself, send everything over OTLP through
OpenTelemetry Collectors.

## What it sends

| Dataset         | What                                                                                 | Sent by                    |
| --------------- | ------------------------------------------------------------------------------------ | -------------------------- |
| `traces`        | Spans from every demo service                                                        | The demo's Collector       |
| `logs`          | The demo services' logs, and the Collector's own                                     | The demo's Collector       |
| `metrics`       | The demo services' metrics, Kafka metrics, and metrics derived from spans            | The demo's Collector       |
| `k8s_events`    | Kubernetes events                                                                    | `cluster-collector`        |
| `k8s_resources` | Kubernetes objects: each change as it happens, and a full listing every 30 minutes  | `cluster-collector`        |
| `k8s_metrics`   | Cluster, node, pod, container and host metrics                                       | `cluster-collector`, `node-collector` |

## How it connects to Telemetry Machine

Telemetry Machine takes OTLP over HTTP. Each Collector has one `otlp_http`
exporter per dataset: `X-API-Key` carries an ingest key, and `X-Dataset-Name`
picks the dataset.

```yaml
exporters:
  otlp_http/k8s_events:
    endpoint: https://api.telemetrymachine.com
    headers:
      X-API-Key: ${env:TM_INGEST_KEY}
      X-Dataset-Name: k8s_events
```

The ingest key never enters Git. You put it in one Secret,
`flux-system/telemetry-machine`. Flux copies it into a Secret in each
Collector's namespace (see `postBuild.substituteFrom` in
[`clusters/demo`](clusters/demo)), and each Collector reads it into
`TM_INGEST_KEY`.

See [Send OpenTelemetry Data](https://telemetrymachine.com/docs/send-opentelemetry)
for the details.

## Layout

| Path                                        | What                                                                                  |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| `clusters/demo/flux-system`                 | Flux itself, and the GitRepository that points it at this repository                  |
| `clusters/demo/*.yaml`                      | The two Flux Kustomizations: `infrastructure`, then `apps`                            |
| `infrastructure/opentelemetry-collector`    | `cluster-collector` (events, objects, cluster metrics) and `node-collector` (kubelet and host metrics) |
| `infrastructure/vertical-pod-autoscaler`    | The VPA recommender, which writes resource advice into each VerticalPodAutoscaler     |
| `apps/otel-demo`                            | The OpenTelemetry Demo and its Collector                                              |

## Versions

| Component                       | Version                       |
| ------------------------------- | ----------------------------- |
| OpenTelemetry Demo              | 3.1.0 (chart 0.42.6)          |
| OpenTelemetry Collector         | 0.164.0 (chart 0.178.1)       |
| Flux                            | 2.10.0                        |
| Vertical Pod Autoscaler         | 1.8.1 (chart 0.15.0)          |

## Run it yourself

You need a Kubernetes cluster with about 8 GB of memory free, and a Telemetry
Machine organization.

1. In Telemetry Machine, create the six datasets above, and an ingest key that
   can write to them.
2. Fork this repository, and change the `url` in
   [`clusters/demo/flux-system/gotk-sync.yaml`](clusters/demo/flux-system/gotk-sync.yaml)
   to your fork.
3. Install Flux, store the ingest key, and point Flux at your fork:

   ```bash
   kubectl apply --server-side -f clusters/demo/flux-system/gotk-components.yaml
   kubectl -n flux-system create secret generic telemetry-machine \
     --from-literal=TM_INGEST_KEY=<your ingest key>
   kubectl apply -f clusters/demo/flux-system/gotk-sync.yaml
   ```

Flux installs the Collectors, then the demo. The first traces reach Telemetry
Machine a few minutes later.

## License

[MIT](LICENSE)
