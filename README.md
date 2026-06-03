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

# MQTT credentials — leave empty if no authentication is required
touch secrets/mqtt_user.txt
touch secrets/mqtt_password.txt
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

The health endpoints should be reachable on any of the configured `externalIPs` at port `8808`:
- `/health/live`
- `/health/ready`

### 5. Remove

```bash
kubectl delete -k .
```

## Configuration

### Environment variables

All runtime configuration is provided through environment variables. Variables with a documented default are optional. Variables without a default are optional (the corresponding feature is disabled when unset) unless marked **Required**.

Sensitive values (`PCONXT_POSTGRES_DSN`, `PCONXT_TESLA_VEHICLE_SEED_JSON`, `PCONXT_API_BEARER_TOKEN_FILE`) should be sourced from the `pconxt-secret` Kubernetes Secret rather than set directly in the Deployment manifest.

| Variable | Required | Default | Description |
|---|---|---|---|
| `PCONXT_HTTP_ADDR` | — | `:8080` | HTTP server bind address (e.g. `:8808`) |
| `PCONXT_ENV` | — | `development` | Deployment environment (`development` or `production`) |
| `PCONXT_CONSOLE_LOG_LEVEL` | — | `debug` | Console log level (`trace`, `debug`, `info`, `warn`, `error`) |
| `PCONXT_POSTGRES_LOG_LEVEL` | — | `warn` | PostgreSQL log level (`trace`, `debug`, `info`, `warn`, `error`) |
| `PCONXT_TIME_ZONE` | — | `America/Denver` | IANA time zone name for the application runtime |
| `PCONXT_POSTGRES_DSN` | No¹ | — | PostgreSQL connection string (`postgres://` or `postgresql://` URI). When unset, persistence is disabled and the application runs in local-only mode. |
| `PCONXT_API_BEARER_TOKEN_FILE` | — | — | Path to a mounted file containing the API bearer token. When unset, API authentication is disabled. |
| `PCONXT_POWER_METRICS_URL` | — | — | HTTPS endpoint for solar/power telemetry polling. When unset, the poller is disabled. |
| `PCONXT_POWER_METRICS_POLL_INTERVAL` | — | `60s` | Power metrics poll interval (Go duration, e.g. `30s`, `65s`). Valid range: `5s`–`5m`. |
| `PCONXT_POWER_METRICS_STALE_AFTER` | — | `2 × poll interval` | How old a snapshot may be before it is reported stale. Must be at least the poll interval and at most `30m`. |
| `PCONXT_TESLA_VEHICLE_SEED_JSON` | — | — | JSON array of vehicle-to-VIN mappings: `[{"name":"myCar","vin":"..."}]`. When unset or empty, Tesla vehicle mapping is disabled. |
| `PCONXT_TESLA_ACTIVE_POLL_INTERVAL` | — | `60s` | How often to poll Tesla charge state during active charging. Must be between `60s`–`10m` and a multiple of `30s`. |
| `PCONXT_TESLA_STALE_RECOVERY_INTERVAL` | — | `6h` | How often to perform stale-data recovery refresh. Must be a whole-second duration between `1h`–`24h`. |
| `PCONXT_TESLA_WAKE_SETTLE_DELAY` | — | `60s` | Delay between wake and read-state-charge during stale-data recovery. Must be a whole-second duration between `10s`–`2m`. |
| `PCONXT_MQTT_HOST` | — | — | Tesla BLE MQTT broker hostname (no scheme, port, or path). When unset, MQTT is disabled. |
| `PCONXT_MQTT_PORT` | — | `1883` | MQTT broker port. Valid range: `1`–`65535`. |
| `PCONXT_MQTT_CLIENT_ID` | — | `PCONXT-TeslaBLE-1` | Stable MQTT client identifier. Must not be empty. |
| `PCONXT_MQTT_TLS_MODE` | — | `disabled` | MQTT transport security (`disabled` or `enabled`). |
| `PCONXT_MQTT_USERNAME_SECRET_REF` | — | — | MQTT username secret reference. Must be paired with `PCONXT_MQTT_PASSWORD_SECRET_REF`. When both are unset, MQTT uses no authentication. |
| `PCONXT_MQTT_PASSWORD_SECRET_REF` | — | — | MQTT password secret reference. Must be paired with `PCONXT_MQTT_USERNAME_SECRET_REF`. |
| `PCONXT_FORECAST_SLEEP_ENABLED` | — | `false` | Master switch for forecast-driven sleep mode (`true` or `false`). |
| `PCONXT_FORECAST_SLEEP_BASE_URL` | — | — | HTTP endpoint for the solar irradiance forecast service. Only `http://` is supported in v1; `https://` makes the feature unavailable. |
| `PCONXT_FORECAST_SLEEP_AM_IRRADIANCE_THRESHOLD` | — | `800` | AM solar irradiance cutoff in W/m². Must be a positive integer. |
| `PCONXT_FORECAST_SLEEP_PM_IRRADIANCE_THRESHOLD` | — | `800` | PM solar irradiance cutoff in W/m². Must be a positive integer. |

¹ `PCONXT_POSTGRES_DSN` is required for durable persistence (policy storage, migration tracking). The application runs without it but features that require persistence will be degraded or unavailable.

### Probes

Both liveness and readiness probes are configured:
- **Liveness**: `GET /health/live`, initial delay 10s, period 30s
- **Readiness**: `GET /health/ready`, initial delay 5s, period 15s

### Service exposure

The Service uses `NodePort` with static `externalIPs` for LAN access. To switch to `ClusterIP` or `LoadBalancer`, edit `pconxt-svc.yaml` and change `spec.type`.

### Persistent storage

A `persistentVolumeClaim` named `synology-k8s-pv02` is mounted at `/data` (subPath `pconxt/data`). Adjust the claim name or remove the volume if no persistent storage is needed.
