# optel-tutotrial

This repository contains a minimal OpenTelemetry setup for Kubernetes using the OpenTelemetry Operator, a sample OpenTelemetryCollector CR, Instrumentation CR, and RBAC manifests. Use these files to deploy a basic tracing pipeline inside your cluster.

## Overview

- `opentelemetry-operator.yaml`: Operator manifests (CRDs, RBAC, ServiceAccount, Deployment, webhook, etc.).
- `opentelemetry-collector.yaml`: OpenTelemetryCollector CR (Collector config, receivers, processors, exporters).
- `opentelemetry-instrumentation.yaml`: Instrumentation CR to auto-inject Java agent environment variables for apps.
- `opentelemetry-rbac.yaml`: RBAC and ServiceAccount for the Collector.
<!-- demo removed from this repository -->

---

## Prerequisites

- Kubernetes cluster (1.21+ recommended) with `kubectl` configured
- Optionally, `cert-manager` if you want to use operator webhooks as-is; the operator YAML references cert-manager for serving TLS: install cert-manager if missing
- Make sure you have permissions to create CRDs and cluster-scoped RBAC

Install cert-manager (optional but recommended):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Quick variables

Pick values for the following variables (example shown):

```bash
NS=otel-demo
COLLECTOR_NAME=optel
INSTR_NAME=otel-inst
```

You can use the demo namespace or pick your own; the instructions below use `NS` as the app namespace.

---

## Step-by-step setup

1) Install the OpenTelemetry operator (CRDs + operator):

```bash
kubectl apply -f opentelemetry-operator.yaml

# wait for the operator to be available
kubectl wait --for=condition=available deployment/opentelemetry-operator-controller-manager -n opentelemetry-operator-system --timeout=120s
kubectl get pods -n opentelemetry-operator-system
```

2) Create your application namespace:

```bash
kubectl create namespace $NS
```

3) Apply RBAC and ServiceAccount for the collector

The file `opentelemetry-rbac.yaml` contains a `<ns-app>` placeholder. Replace it with your namespace before applying:

```bash
NS=otel-demo
sed "s|<ns-app>|$NS|g" opentelemetry-rbac.yaml | kubectl apply -f -
```

4) Apply an OpenTelemetryCollector CR

Edit or replace placeholders in `opentelemetry-collector.yaml` before applying. The common placeholders are:

- `<name-collector>`: Collector CR name (e.g., `optel` -> service becomes `optel-collector`)
- `<ns-app>`: The namespace where the Collector CR should be created
- `<endpoint-traces-dataSource-svc-port-4317>` and `<endpoint-traces-dataSource-svc-port-4318>`: If you have a backend (e.g., Tempo, Jaeger), fill the endpoint there. For a minimal setup, you can keep `debug` enabled instead of external exporters.

Example to create the collector and keep debug exporter for simple visibility:

```bash
COLLECTOR_NAME=${COLLECTOR_NAME:-optel}
NS=${NS:-otel-demo}
sed -e "s|<name-collector>|$COLLECTOR_NAME|g" \
	-e "s|<ns-app>|$NS|g" \
	-e "s|<endpoint-traces-dataSource-svc-port-4317>|tempo-collector.tempo.svc.cluster.local:4317|g" \
	-e "s|<endpoint-traces-dataSource-svc-port-4318>|http://tempo-collector.tempo.svc.cluster.local:4318|g" \
	opentelemetry-collector.yaml | kubectl apply -f -
```

Notes:
- Replace the `tempo-collector` values with your desired backend. If you don't have a backend, the `debug` exporter in the YAML will still print traces to Collector logs, which is useful for initial testing.

5) Apply the Instrumentation CR

This CR injects Java agent env vars into applications in the namespace you specified.

```bash
sed -e "s|<name-instrumentation>|$INSTR_NAME|g" \
	-e "s|<ns-app>|$NS|g" \
	-e "s|<endpoint-collector-svc-port-4318>|http://$COLLECTOR_NAME-collector.$NS.svc.cluster.local:4318|g" \
	opentelemetry-instrumentation.yaml | kubectl apply -f -
```

6) Deploy your instrumented app

Deploy your own app or any existing application in the namespace `NS`. Ensure it is instrumented or that the auto-instrumentation injection is enabled by the `Instrumentation` CR.

7) Verify resources and traces

```bash
# Verify OpenTelemetry custom resources
kubectl get opentelemetrycollectors -n $NS
kubectl get instrumentations -n $NS

# Look at pods and logs
kubectl get pods -n $NS
kubectl logs -n $NS <collector-pod-name> -c otel-collector

# For instrumented app pods, check that the OTel Java agent environment variables were injected
kubectl describe pod -n $NS <app-pod-name>

# If you configured an external exporter (Tempo, Jaeger, etc.), use your backend UI to check the traces
```

---

## Tips and troubleshooting

- If you are using operator webhooks, ensure `cert-manager` is installed and the webhook certificates are valid.
- If pods do not receive the instrumentation environment variables, check the operator logs:

```bash
kubectl logs -n opentelemetry-operator-system deployment/opentelemetry-operator-controller-manager
```

- If Collector exports are not visible in your backend: check the exporter endpoints in `opentelemetry-collector.yaml` and that the backend is reachable from the cluster.
- If Collector is not running or CrashLooping: `kubectl describe pod -n $NS <collector-pod-name>` and inspect container logs.

---

## Cleanup

```bash
# Remove your application
# kubectl delete -f <your-app.yaml> -n $NS
# Delete instrumentation
sed -e "s|<name-instrumentation>|$INSTR_NAME|g" -e "s|<ns-app>|$NS|g" opentelemetry-instrumentation.yaml | kubectl delete -f -
# Delete collector and rbac
sed -e "s|<name-collector>|$COLLECTOR_NAME|g" -e "s|<ns-app>|$NS|g" opentelemetry-collector.yaml | kubectl delete -f -
sed "s|<ns-app>|$NS|g" opentelemetry-rbac.yaml | kubectl delete -f -
# Optionally, delete namespace
kubectl delete namespace $NS
```
