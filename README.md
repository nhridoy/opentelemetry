# Observability Stack (OpenTelemetry + Grafana LGTM + Pyroscope)

A self-contained Docker Compose stack that receives traces, metrics, logs and
profiles from an application (e.g. a Django + Celery backend running on a
**different server**) and lets you explore them in Grafana.

```
 Django / Celery server                      Observability server
┌──────────────────────┐   HTTPS + basic   ┌──────────────────────────────────────────────┐
│ OTel SDK             │ ───────────────►  │ otel-collector ─┬─► Tempo      (traces)      │
│ (traces/metrics/logs)│  otel.<domain>    │                 ├─► Prometheus (metrics)     │
│                      │                   │                 └─► Loki       (logs)        │
│ Pyroscope SDK        │ ───────────────►  │ pyroscope-auth (nginx) ─► Pyroscope (profiles)│
│ (profiles)           │ profiles.<domain> │                                              │
└──────────────────────┘                   │ Grafana ◄── queries all four datasources     │
                                           └──────────────────────────────────────────────┘
                                                      grafana.<domain>
```

## What's in the repo

| Path | Purpose |
|---|---|
| `docker-compose.observability.yml` | The whole stack (7 services + a one-shot volume initializer) |
| `.env.example` | Template for the required secrets/config |
| `observability/otel-collector.yaml` | Collector: OTLP/HTTP receiver with basic auth, drops noisy Redis/`set_config` root spans, strips query strings from URLs (may contain tokens), routes to Tempo/Prometheus/Loki |
| `observability/tempo.yaml` | Tempo (traces), 7-day retention, TraceQL metrics enabled |
| `observability/loki.yaml` | Loki (logs) via native OTLP endpoint, 7-day retention |
| `observability/prometheus.yml` | Prometheus (metrics) via OTLP receiver, 30-day retention (set in compose) |
| `observability/grafana-datasources.yaml` | Auto-provisioned datasources with trace ↔ log links |
| `observability/pyroscope-auth/` | nginx basic-auth proxy in front of Pyroscope (which has no login of its own) |

### Services

| Service | Image | Internal port | Exposed publicly? |
|---|---|---|---|
| `otel-collector` | otel/opentelemetry-collector-contrib | **4318** (OTLP/HTTP) | Yes, via domain |
| `pyroscope-auth` | nginx (→ Pyroscope :4040) | **80** | Yes, via domain |
| `grafana` | grafana/grafana | **3000** | Yes, via domain |
| `tempo` | grafana/tempo | 3200 (HTTP), 4317 (OTLP gRPC) | No |
| `loki` | grafana/loki | 3100 | No |
| `prometheus` | prom/prometheus | 9090 | No |
| `pyroscope` | grafana/pyroscope | 4040 | No (only via `pyroscope-auth`) |

**No host ports are published** by the compose file. Everything is reached through
a reverse proxy (Dokploy/Traefik) attached to the Docker network.

## Hosting

### 1. Configure

```bash
cp .env.example .env
# edit .env and fill in every value
```

| Variable | Required | Description |
|---|---|---|
| `OTLP_USERNAME` / `OTLP_PASSWORD` | yes | Basic-auth credentials the app uses to send telemetry to the Collector |
| `PYROSCOPE_USERNAME` / `PYROSCOPE_PASSWORD` | yes | Basic-auth credentials for pushing profiles |
| `GRAFANA_ADMIN_USER` | no (default `admin`) | Grafana admin login |
| `GRAFANA_ADMIN_PASSWORD` | yes | Grafana admin password |
| `GRAFANA_ROOT_URL` | no | Public Grafana URL, e.g. `https://grafana.example.com` |
| `CONTAINER_PREFIX` | no (default `lumicore`) | Prefix for container names |

Compose refuses to start if a required value is missing. Use strong random
passwords (e.g. `openssl rand -base64 24`). Avoid `:` in usernames; avoid
characters like `$` in passwords (compose and nginx envsubst may interpret them).

### 2. Start

```bash
docker compose -f docker-compose.observability.yml up -d
docker compose -f docker-compose.observability.yml ps
```

With **Dokploy**: create a Compose application from this repo, set the
variables from `.env` in its Environment tab, deploy, then add the domains below.

### 3. Map domains to ports

Create DNS `A` records pointing at the server, then add these domains (HTTPS /
Let's Encrypt on) in Dokploy → the compose app → Domains:

| Domain | Service | Container port | Used by |
|---|---|---|---|
| `otel.<yourdomain>` | `otel-collector` | **4318** | App sends OTLP traces/metrics/logs here |
| `profiles.<yourdomain>` | `pyroscope-auth` | **80** | App pushes Pyroscope profiles here |
| `grafana.<yourdomain>` | `grafana` | **3000** | You, in the browser |

Do **not** map Tempo, Loki, Prometheus or Pyroscope directly; they have no
authentication. Grafana reaches them over the internal Docker network.

> Using plain nginx/Caddy/Traefik instead of Dokploy? Either attach the proxy to
> the compose network and proxy to the same service:port pairs above, or add
> `ports:` mappings yourself (e.g. `127.0.0.1:4318:4318`) and proxy to localhost.
> Note host port 3000 is taken by Dokploy's own UI, which is why Grafana isn't published.

### 4. Verify

```bash
# Expect 401 without credentials, 200/405 with:
curl -i https://otel.<yourdomain>/v1/traces
curl -i -u "$OTLP_USERNAME:$OTLP_PASSWORD" -X POST https://otel.<yourdomain>/v1/traces \
  -H 'Content-Type: application/json' -d '{}'

curl -i https://profiles.<yourdomain>/            # 401
```

Then log in at `https://grafana.<yourdomain>` with the Grafana admin credentials;
Prometheus, Tempo, Loki and Pyroscope datasources are already provisioned.

## Generating the base64 credentials for OTel (Django, etc.)

OTLP requests use HTTP **Basic auth**: the header is
`Authorization: Basic base64(username:password)`, using the same
`OTLP_USERNAME` / `OTLP_PASSWORD` you put in the observability server's `.env`.

### Generate

```bash
# Linux / macOS  (-n avoids encoding a trailing newline; -w0 avoids line wrapping)
echo -n 'my_otlp_user:my_otlp_password' | base64 -w0      # Linux
echo -n 'my_otlp_user:my_otlp_password' | base64          # macOS

# Using the values from .env
set -a; source .env; set +a
echo -n "$OTLP_USERNAME:$OTLP_PASSWORD" | base64 -w0

# Python (any OS)
python3 -c "import base64; print(base64.b64encode(b'my_otlp_user:my_otlp_password').decode())"
```

Example: `echo -n 'admin:secret' | base64` → `YWRtaW46c2VjcmV0`

> Always use `echo -n`. Without it a newline is encoded and every request fails with 401.

### Use in the Django server's `.env`

```env
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.<yourdomain>
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Basic%20<BASE64_VALUE>
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Pyroscope (separate credentials, handled by the SDK, no base64 needed)
PYROSCOPE_SERVER_ADDRESS=https://profiles.<yourdomain>
PYROSCOPE_USERNAME=<pyroscope_username>
PYROSCOPE_PASSWORD=<pyroscope_password>
```

Notes:

- The space after `Basic` must be URL-encoded as `%20` in the env var
  (the OTel spec requires percent-encoded header values). Don't put quotes around the value.
- If the base64 string ends in `=` padding and your SDK complains, encode it as `%3D`.
- Use the endpoint **without** a `/v1/traces` suffix; the SDK appends per-signal
  paths (`/v1/traces`, `/v1/metrics`, `/v1/logs`) itself.
- Set `OTEL_SERVICE_NAME` (e.g. `backend`, `celery-worker`, `celery-beat`) and
  `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production` so telemetry is labelled in Grafana.

### Docker / docker-compose on the Django side

```yaml
environment:
  OTEL_EXPORTER_OTLP_ENDPOINT: https://otel.example.com
  OTEL_EXPORTER_OTLP_HEADERS: "Authorization=Basic%20YWRtaW46c2VjcmV0"
```

### Setting it in code instead (Python)

```python
import base64, os
token = base64.b64encode(
    f"{os.environ['OTLP_USERNAME']}:{os.environ['OTLP_PASSWORD']}".encode()
).decode()

from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(
    endpoint="https://otel.example.com/v1/traces",   # full path when passed explicitly
    headers={"Authorization": f"Basic {token}"},     # a real space here, not %20
)
```

### Run Django with auto-instrumentation

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
opentelemetry-instrument gunicorn config.wsgi:application   # or celery -A config worker
```

## Operations

```bash
# Logs
docker compose -f docker-compose.observability.yml logs -f otel-collector

# Update after changing a config file in observability/
docker compose -f docker-compose.observability.yml up -d --force-recreate otel-collector

# Rotate credentials: edit .env, then recreate the affected service
docker compose -f docker-compose.observability.yml up -d --force-recreate otel-collector pyroscope-auth grafana
# ...and regenerate the base64 value + update the Django server's .env.
```

Data lives in named volumes (`tempo_data`, `loki_data`, `prometheus_data`,
`pyroscope_data`, `grafana_data`). Retention: traces 7d, logs 7d, metrics 30d.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| App gets `401 Unauthorized` | Wrong base64, missing `echo -n`, or credentials differ from the server's `OTLP_*` values |
| App gets `404` | Endpoint includes a path suffix the SDK also appends, or domain isn't mapped to port 4318 |
| Compose says `set X in .env` | A required variable is empty |
| Grafana "empty ring" in Traces Drilldown | Tempo `metrics_generator` config missing/changed |
| Tempo/Loki/Pyroscope permission errors | `volume-init` didn't finish; check `docker compose ps -a` |
