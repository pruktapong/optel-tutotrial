# ⚠️ AI-generated
---
# OpenTelemetry demo: Collector + Instrumentation + Tempo

This demo shows multiple ways to deploy an OpenTelemetry Collector and instrument a Java app. Choose whichever pattern best matches your environment:

- Collector as a DaemonSet (central collector running on every node) with a Service: `01-collector-daemonset.yaml`
- Collector as a CR-managed Deployment using the OpenTelemetry Operator: `01-collector-java-deployment.yaml`
- Collector as a sidecar managed by the Operator (mode: `sidecar`): `01-collector-java-sidecar.yaml`
- Instrumentation options for Java (auto-injection):
	- `02-instrumentation-java-daemonset.yaml` — use when collector is a central DaemonSet or separate Deployment and you want the agent to call it by service.
	- `02-instrumentation-java-deployment.yaml` — instrumentation for deployment-based environments.
	- `02-instrumentation-java-sidecar.yaml` — agent sends to `localhost` (i.e., a per-pod sidecar collector)
- Example Java application + service: `app.yaml` (includes `instrumentation.opentelemetry.io/inject-java: "true"` annotation)
- Tempo backend test deployment: `tempo/tempo.yaml` in `monitoring` namespace
- Namespace/RBAC for the demo: `00-ns.yaml`

This README covers:
- How to set up Tempo (backend)
- How to deploy the collector variants
- How to deploy and validate the Java app with instrumentation
- Quick troubleshooting and checks

---

## Prerequisites

- A Kubernetes cluster accessible via `kubectl`
- Optionally the OpenTelemetry Operator (if you plan to use `OpenTelemetryCollector` CRs), which the operator YAML would install (not included here).
- `nc`, `ntp`, `jq` or `kubectl` for debugging and validation

---

## Apply Namespace and RBAC

This demo uses the `java-test` namespace. Apply the namespace and RBAC first:

```bash
kubectl apply -f 00-ns.yaml
kubectl get ns java-test -o wide
```

If your environment has finalizer/namespace Terminating issues, resolve them before proceeding.

---

## Deploy Tempo (backend)

This tutorial includes a minimal Tempo deployment used as a backend for traces. If you already have a Tempo stack, skip this step.

```bash
kubectl apply -f tempo/tempo.yaml
kubectl -n monitoring get pods,svc -o wide
```

Tempo will accept OTLP on ports 4317 (gRPC) and 4318 (HTTP/Protobuf) in the `monitoring` namespace.

---

## Option A: Deploy Collector as a DaemonSet (useful for node local collectors)

This approach runs one collector per node and exposes a ClusterIP `otel-collector` service that all pods can reach.

```bash
kubectl apply -f 01-collector-daemonset.yaml
kubectl -n java-test get ds,deploy,svc -l app=otel-collector -o wide
```

Collector config mounts a `ConfigMap` named `otel-collector-config` and forwards traces to `tempo.observability.svc:4317` and `tempo.observability.svc:4318`. Confirm the service in `java-test` namespace: `otel-collector`.

---

## Option B: Deploy Collector via OpenTelemetry Operator (Deployment mode)

If you have the OpenTelemetry Operator installed, you can create a managed Collector with `01-collector-java-deployment.yaml`.

Note: Deploying this resource requires the operator to be running in the cluster.

```bash
kubectl apply -f 01-collector-java-deployment.yaml
kubectl -n java-test get opentelemetrycollector
kubectl -n java-test get pods,svc -l app.kubernetes.io/name=opentelemetry-collector -o wide
```

If the operator cannot create resources (e.g., because of namespace Terminating), resolve that first.

---

## Option C: Collector per-pod (Sidecar mode via Operator)

Use `01-collector-java-sidecar.yaml` (OpenTelemetryCollector CR with `mode: sidecar`).

This instructs the operator to create a sidecar in pods where the collector should be added. With sidecar mode, instrumentation may export to `localhost`.

```bash
kubectl apply -f 01-collector-java-sidecar.yaml
```

---

## Instrumentation options (choose one)

These CRs provide `OTEL_*` env vars for Java auto-instrumentation.

- When collector is a centralized DaemonSet or Deployment (service `otel-collector`), use the 'deployment' or 'daemonset' instrumentation: `02-instrumentation-java-deployment.yaml` or `02-instrumentation-java-daemonset.yaml`. They point to `http://otel-collector.java-test.svc:4318`.
- When collector is a sidecar (in pod), use the 'sidecar' instrumentation `02-instrumentation-java-sidecar.yaml` which sets `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` (or gRPC 4317 if you prefer gRPC).

Apply the instrumentation CR based on your chosen strategy:

```bash
# For centralized collector
kubectl -n java-test apply -f 02-instrumentation-java-deployment.yaml

# For sidecar
kubectl -n java-test apply -f 02-instrumentation-java-sidecar.yaml
```

---

## Deploy the demo Java application

By default `app.yaml` is annotated for automatic Java agent injection. If you configured instrumentation in `java-test`, this will inject the agent with the `OTEL_*` env vars.

```bash
kubectl -n java-test apply -f app.yaml
kubectl -n java-test get pods -l app=my-java -o wide
```

Check the pod to ensure the agent is injected and `OTEL_*` env variables are present:

```bash
POD=$(kubectl -n java-test get pods -l app=my-java -o jsonpath='{.items[0].metadata.name}')
kubectl -n java-test exec -it $POD -- printenv | grep OTEL || true
kubectl -n java-test describe pod $POD | sed -n '1,200p'
```

Look for the `-javaagent` argument in the container command line (or for agent container if instrumented as a sidecar), depending on the instrumentation mechanism.

---

## Basic validation

1) Confirm the collector service & pods are present:

```bash
kubectl -n java-test get pods,svc -o wide
```

2) From the application pod, test connectivity to the collector HTTP endpoint (if using HTTP/protobuf):

```bash
kubectl -n java-test exec -it $POD -- nc -vz otel-collector.java-test.svc 4318 || true
kubectl -n java-test exec -it $POD -- nc -vz otel-collector.java-test.svc 4317 || true
```

3) Inspect application logs for exporter messages (e.g., `Failed to export` vs `Established connection`):

```bash
kubectl -n java-test logs $POD | grep -E 'Failed to export|Failed to connect|Exporter failed|Established|Connected' || true
```

4) Check collector logs for inbound OTLP and forwarding to Tempo:

```bash
kubectl -n java-test logs $(kubectl -n java-test get pods -l app=otel-collector -o jsonpath='{.items[0].metadata.name}') | tail -n 200
```

5) Confirm traces appear in Tempo UI (if you deployed tempo) or on the collector logs.

---

## Load test & instrumentation check (optional)

The demo includes a `05-load-test.yaml.bak` - this can be converted to a load Job/Deployment to generate traffic for the demo app to create spans that will be forwarded to Tempo.

---

## Troubleshooting

- Agent logs `Failed to connect to localhost:4317`:
	- This happens when the agent currently attempts to use `localhost` because instrumentation points to a local endpoint but no sidecar is present.
	- Fix: either deploy a sidecar collector or set your instrumentation env to point to the service `otel-collector.java-test.svc:4318` and `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`.

- Operator cannot create resources (namespace `Terminating`):
	- Fix namespace finalizers or re-create namespace; see steps in the main tutorial on finalizers.

- Collector not forwarding to Tempo:
	- Check `exporters:` configuration in the collector YAML and that `service.pipelines` include the exporter(s) forwarding to Tempo.

- B3 propagation issues (`Invalid TraceId in B3 header`):
	- Validate clients and proxies; prefer `tracecontext` if using W3C traces.
	- You can change `OTEL_PROPAGATORS` in the instrumentation to `tracecontext,baggage`.

---

## Cleanup

To remove all demo artifacts:

```bash
kubectl -n java-test delete -f 01-collector-daemonset.yaml || true
kubectl -n java-test delete -f 01-collector-java-deployment.yaml || true
kubectl -n java-test delete -f 01-collector-java-sidecar.yaml || true
kubectl -n java-test delete -f 02-instrumentation-java-deployment.yaml || true
kubectl -n java-test delete -f 02-instrumentation-java-daemonset.yaml || true
kubectl -n java-test delete -f 02-instrumentation-java-sidecar.yaml || true
kubectl -n java-test delete -f app.yaml || true
kubectl -n monitoring delete -f tempo/tempo.yaml || true
kubectl delete -f 00-ns.yaml || true
```

---

If you'd like, I can:
- Generate a small `install.sh` that deploys a chosen pattern (DaemonSet, operator deployment, or sidecar) and performs validation checks.
- Add more comments inside the YAML files explaining `OTEL_*` env vars and how to set gRPC vs HTTP/protobuf.

---

⚠️ Note: This demo uses unsecured HTTP for OTLP to Tempo and has `tls: insecure` in the collector. Use TLS and authentication for production.

---

## ⚠️ Container Images ⚠️

Here are the container images used in this demo:

- **grafana/tempo:2.6.0**  
	- Grafana Tempo distributed tracing backend. Stores and queries traces (backend for Jaeger/OTLP data). Version 2.6.0.

- **otel/opentelemetry-collector-contrib:0.102.0**  
	- OpenTelemetry Collector (contrib build) — community receivers, exporters, processors and extensions bundled for collecting and exporting telemetry. Version 0.102.0.

- **ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:0.140.0**  
	- Official OpenTelemetry Collector core release (minimal, upstream-built collector binary) for receiving, processing and exporting telemetry. Version 0.140.0.

- **ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.33.6**  
	- Java auto-instrumentation agent image provided by OpenTelemetry tooling/operator to automatically instrument Java applications for tracing/metrics. Version 1.33.6.

- **ghcr.io/open-telemetry/opentelemetry-operator/opentelemetry-operator:0.140.0**  
	- Kubernetes operator to deploy and manage OpenTelemetry components (Collectors, auto-instrumentation, CRDs). Version 0.140.0.

- **quay.io/jetstack/cert-manager-webhook:v1.19.1**  
	- cert-manager webhook component that serves admission webhooks used by cert-manager CRDs. Version v1.19.1.

- **quay.io/jetstack/cert-manager-cainjector:v1.19.1**  
	- cert-manager CA injector that injects CA bundles into webhook/validating webhook configurations and secrets. Version v1.19.1.

- **quay.io/jetstack/cert-manager-controller:v1.19.1**  
	- cert-manager controller that reconciles Certificate, Issuer and related CRDs to provision TLS certificates. Version v1.19.1.