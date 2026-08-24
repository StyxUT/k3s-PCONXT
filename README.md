# k3s-PCONXT

Kubernetes manifests to deploy [PCONXT](https://github.com/StyxUT/PCONXT) (Power Consumption Optimizer Next) on a K3s cluster.

## Overview

This project provides the Kubernetes resources needed to run PCONXT — a Go application for optimizing residential electrical power consumption by coordinating controllable loads against available power conditions and operational policies. The application code lives in the [styxut/PCONXT](https://github.com/StyxUT/PCONXT) repository. The manifest pins an immutable revision tag rather than the mutable `latest` tag.

## Manifests

| File | Kind | Description |
|------|------|-------------|
| `pconxt-deployment.yaml` | `Deployment` | Single-replica deployment of `styxut/pconxt:git-537a852f8162` in the `default` namespace. Uses a `Recreate` strategy, listens on port `8866`, and configures startup, liveness, and readiness probes. |
| `pconxt-svc.yaml` | `Service` | Exposes port `8866` through a `ClusterIP` Service and the available cluster-node IPs for HAProxy. |
| `kustomization.yaml` | `Kustomization` | Kustomize overlay that bundles the Deployment and Service, generates the `pconxt-secret` from files in `secrets/`, and disables the name-suffix hash. |
| `pconxt.env.example` | Example configuration | Sanitized reference containing every application setting represented by the live Deployment. |

## Prerequisites

- A K3s (or any Kubernetes) cluster
- `kubectl` configured for the target cluster
- (Optional) `kustomize` or `kubectl` v1.14+

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
2. Create the `pconxt` Deployment with one replica in the `default` namespace.
3. Create the `pconxt` Service on port `8866`, bound to the configured `externalIPs`.

### 4. Verify

```bash
kubectl get pods -l app=pconxt
kubectl get svc pconxt
```

The health endpoints should be reachable over HTTP on any configured `externalIP` at port `8866`:
- `/health/live`
- `/health/ready`

For example:

```bash
curl --fail http://192.168.0.221:8866/health/ready
```

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
| `PCONXT_HTTP_ADDR` | — | `:8080` | HTTP server bind address (e.g. `:8866`) |
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
| `PCONXT_VENSTAR_THERMOSTAT_SEED_JSON` | — | — | JSON array of thermostat policy and Venstar mapping bootstrap entries. Existing durable records remain authoritative. |
| `PCONXT_THERMOSTAT_COOLING_SIGMOID_T` | — | `0.25` | Thermostat cooling consumer-type sigmoid midpoint. |
| `PCONXT_THERMOSTAT_COOLING_SIGMOID_K` | — | `1` | Thermostat cooling consumer-type sigmoid steepness. |
| `PCONXT_VEHICLE_CHARGING_SIGMOID_T` | — | `1` | Vehicle charging consumer-type sigmoid midpoint. |
| `PCONXT_VEHICLE_CHARGING_SIGMOID_K` | — | `1` | Vehicle charging consumer-type sigmoid steepness. |
| `PCONXT_TESLA_ACTIVE_POLL_INTERVAL` | — | `60s` | How often to poll Tesla charge state during active charging. Must be between `60s`–`10m` and a multiple of `30s`. |
| `PCONXT_TESLA_STALE_RECOVERY_INTERVAL` | — | `6h` | How often to perform stale-data recovery refresh. Must be a whole-second duration between `1h`–`24h`. |
| `PCONXT_TESLA_WAKE_SETTLE_DELAY` | — | `60s` | Delay between wake and read-state-charge during stale-data recovery. Must be a whole-second duration between `10s`–`2m`. |
| `PCONXT_MQTT_HOST` | — | — | Tesla BLE MQTT broker hostname (no scheme, port, or path). When unset, MQTT is disabled. |
| `PCONXT_MQTT_PORT` | — | `1883` | MQTT broker port. Valid range: `1`–`65535`. |
| `PCONXT_MQTT_CLIENT_ID` | — | `PCONXT-TeslaBLE-1` | Stable MQTT client identifier. Must not be empty. |
| `PCONXT_MQTT_TLS_MODE` | — | `disabled` | MQTT transport security (`disabled` or `enabled`). |
| `PCONXT_MQTT_USERNAME_SECRET_REF` | — | — | MQTT username secret reference. Must be paired with `PCONXT_MQTT_PASSWORD_SECRET_REF`. When both are unset, MQTT uses no authentication. |
| `PCONXT_MQTT_PASSWORD_SECRET_REF` | — | — | MQTT password secret reference. Must be paired with `PCONXT_MQTT_USERNAME_SECRET_REF`. |
| `PCONXT_VENSTAR_TRUSTED_CIDRS` | — | — | JSON array of CIDRs containing permitted Venstar endpoint addresses. |
| `PCONXT_VENSTAR_TRUSTED_HOSTNAMES` | — | — | JSON array of exact permitted Venstar endpoint hostnames. Hostname addresses must also be within a trusted CIDR. |
| `PCONXT_FORECAST_SLEEP_ENABLED` | — | `false` | Master switch for forecast-driven sleep mode (`true` or `false`). |
| `PCONXT_FORECAST_SLEEP_BASE_URL` | — | — | HTTP endpoint for the solar irradiance forecast service. Only `http://` is supported in v1; `https://` makes the feature unavailable. |
| `PCONXT_FORECAST_SLEEP_AM_IRRADIANCE_THRESHOLD` | — | `800` | AM solar irradiance cutoff in W/m². Must be a positive integer. |
| `PCONXT_FORECAST_SLEEP_PM_IRRADIANCE_THRESHOLD` | — | `800` | PM solar irradiance cutoff in W/m². Must be a positive integer. |

¹ `PCONXT_POSTGRES_DSN` is required for durable persistence (policy storage, migration tracking). The application runs without it but features that require persistence will be degraded or unavailable.

### Probes and HAProxy

The Kubernetes probes are:

- **Startup**: `GET /health/ready`, allowing up to 120 seconds for startup and database migrations
- **Liveness**: `GET /health/live`, every 30 seconds
- **Readiness**: `GET /health/ready`, every 10 seconds

HAProxy should terminate TLS for `https://pconxt.styxut.net` and forward plain HTTP to port `8866`. Use `GET /health/ready` as the backend health check and require an HTTP `200` response. Available backend addresses are:

- `192.168.0.211:8866`
- `192.168.0.221:8866`
- `192.168.0.222:8866`
- `192.168.0.223:8866`

Example HAProxy backend:

```haproxy
backend pconxt
    option httpchk GET /health/ready
    http-check expect status 200
    server k3s-mgmt-01 192.168.0.211:8866 check
    server k3s-worker-01 192.168.0.221:8866 check
    server k3s-worker-02 192.168.0.222:8866 check
    server k3s-worker-03 192.168.0.223:8866 check
```

### Service exposure

The `ClusterIP` Service also declares static `externalIPs` for direct HAProxy access on service port `8866`. It does not allocate a separate Kubernetes `NodePort`.

### Persistent storage

PCONXT stores durable application state in PostgreSQL and does not require a pod-mounted persistent volume. PostgreSQL storage and backups are managed separately.
