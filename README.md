# k3s-PCONXT

Kubernetes manifests to deploy [PCONXT](https://github.com/StyxUT/PCONXT) (Power Consumption Optimizer Next) on a K3s cluster.

## Overview

This project provides the Kubernetes resources needed to run PCONXT — a Go application for optimizing residential electrical power consumption by coordinating controllable loads against available power conditions and operational policies. The application code lives in the [styxut/PCONXT](https://github.com/StyxUT/PCONXT) repository and is published as the Docker image `styxut/pconxt:latest`.

## Manifests

| File | Kind | Description |
|------|------|-------------|
| `pconxt-deployment.yaml` | `Deployment` | Single-replica deployment of `styxut/pconxt:latest`. Uses a `Recreate` strategy. Exposes container port `8080` (TCP). Configures liveness and readiness probes against `/health/live` and `/health/ready`. Injects sensitive configuration (PostgreSQL DSN, Tesla vehicle seed) from the `pconxt-secret` Kubernetes Secret. Mounts `synology-k8s-pv02` at `/data` for persistent storage. |
| `pconxt-svc.yaml` | `Service` | Exposes the deployment on port `8080` with `NodePort` type and static `externalIPs` on the `192.168.0.0/24` subnet (`192.168.0.211`–`192.168.0.224`). |
| `kustomization.yaml` | `Kustomization` | Kustomize overlay that bundles the Deployment and Service, generates the `pconxt-secret` from files in `secrets/`, and disables the name-suffix hash. |

## Prerequisites

- A K3s (or any Kubernetes) cluster
- `kubectl` configured for the target cluster
- (Optional) `kustomize` or `kubectl` v1.14+
- A `synology-k8s-pv02` PersistentVolumeClaim available in the cluster (or adjust the volume claim name)

## Usage

### 1. Prepare secrets

Create the secret files in the `secrets/` directory:

```bash
echo -n 'postgres://user:pass@host:5432/db?sslmode=disable' > secrets/postgres_dsn.txt
echo -n '[{"name":"myCar","vin":"..."}]' > secrets/tesla_vehicle_seed.json
```

> The `secrets/` directory is gitignored and will never be committed.

### 2. Configure environment

Edit `pconxt-deployment.yaml` to match your runtime environment: MQTT broker address, Power Metrics URL, forecast sleep settings, log levels, etc.

### 3. Deploy

```bash
kubectl apply -k .
```

This will:
1. Generate the `pconxt-secret` Kubernetes Secret from the secret files.
2. Create the `pconxt` Deployment with one replica.
3. Create the `pconxt` Service, bound to the configured `externalIPs`.

### 4. Verify

```bash
kubectl get pods -l app=pconxt
kubectl get svc pconxt
```

The health endpoints should be reachable on any of the configured `externalIPs` at port `8080`:
- `/health/live`
- `/health/ready`

### 5. Remove

```bash
kubectl delete -k .
```

## Configuration

### Environment variables

Runtime configuration is provided through environment variables in `pconxt-deployment.yaml`. Key variables include:

| Variable | Default | Description |
|---|---|---|
| `PCONXT_HTTP_ADDR` | `:8080` | HTTP server bind address |
| `PCONXT_CONSOLE_LOG_LEVEL` | `debug` | Console log level |
| `PCONXT_POSTGRES_LOG_LEVEL` | `warn` | PostgreSQL log level |
| `PCONXT_ENV` | `development` | Deployment environment |
| `PCONXT_POSTGRES_DSN` | *(secret)* | PostgreSQL connection string |
| `PCONXT_TESLA_VEHICLE_SEED_JSON` | *(secret)* | JSON vehicle-to-VIN mappings |
| `PCONXT_MQTT_HOST` | *(empty)* | Tesla BLE MQTT broker hostname |
| `PCONXT_MQTT_PORT` | `1883` | MQTT broker port |
| `PCONXT_FORECAST_SLEEP_ENABLED` | `false` | Enable forecast-driven sleep mode |

Sensitive values (`PCONXT_POSTGRES_DSN`, `PCONXT_TESLA_VEHICLE_SEED_JSON`) are sourced from the `pconxt-secret` Kubernetes Secret, generated from files in `secrets/`.

### Probes

Both liveness and readiness probes are configured:
- **Liveness**: `GET /health/live`, initial delay 10s, period 30s
- **Readiness**: `GET /health/ready`, initial delay 5s, period 15s

### Service exposure

The Service uses `NodePort` with static `externalIPs` for LAN access. To switch to `ClusterIP` or `LoadBalancer`, edit `pconxt-svc.yaml` and change `spec.type`.

### Persistent storage

A `persistentVolumeClaim` named `synology-k8s-pv02` is mounted at `/data` (subPath `pconxt/data`). Adjust the claim name or remove the volume if no persistent storage is needed.
