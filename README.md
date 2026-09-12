# Monitoring & Logging Stack

A full observability stack for a homelab environment, orchestrated with
**Docker Compose**: centralized logging (ELK), metrics (Prometheus + Grafana),
distributed tracing (Zipkin), and a cache demo (Redis) — 8 services across
**4 isolated networks**, all debugged hands-on.

![Platform](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Compose](https://img.shields.io/badge/Compose%20v2-2496ED?style=flat&logo=docker&logoColor=white)
![Elastic](https://img.shields.io/badge/ELK%209.5.3-FED10A?style=flat&logo=elastic&logoColor=black)

## Architecture

```
                 ┌─────────────────────────────────┐
  logs (TCP 5010)│           LOGGING               │
 ──────────────► │  Logstash ──► Elasticsearch ◄───┼──── Kibana (5601)
                 │                    ▲            │
                 │                 Zipkin ─────────┼──── (9411, storage=ES)
                 └─────────────────────────────────┘
   ┌──────────────────┐        ┌──────────────┐
   │    MONITORING    │        │     CACHE    │
   │ Prometheus 9090  │        │  Redis 6379  │
   │   Grafana  3000  │        │ RedisInsight │
   └──────────────────┘        └──────────────┘
```

| Service | Port | Network(s) | Notes |
|---|---|---|---|
| Elasticsearch 9.5.3 | 9200/9300 | logging | single-node, healthcheck-gated |
| Logstash | 5010→5000 | logging | JSON-over-TCP input, read-only config mount |
| Kibana | 5601 | logging | `ELASTICSEARCH_HOSTS` via embedded DNS |
| Zipkin | 9411 | logging + tracing | ES as storage backend |
| Prometheus | 9090 | monitoring | self-scrape, config mounted `:ro` |
| Grafana | 3000 | monitoring | named volume for dashboards |
| Redis / RedisInsight | 6379 / 5540 | cache | isolated from logging |

## Quick Start

```bash
docker compose up -d

# verify
docker ps --format "table {{.Names}}\t{{.Status}}"

# end-to-end log pipeline smoke test
echo '{"message":"hello stack"}' | nc localhost 5010
curl -s "localhost:9200/_cat/indices?v"
```

Then open:
- **Kibana** → `:5601` → data view `ds-logs-*` → Discover → see your message
- **Prometheus** → `:9090` → Status → Targets (green `prometheus` job)
- **Grafana** → `:3000` → datasource → `http://prometheus:9090`
- **Zipkin** → `:9411` → query traces

## Layout

```
├── docker-compose.yml
├── logstash/logstash.conf          # tcp/json input → elasticsearch output
└── prometheus/prometheus.yml       # global + scrape_configs
```

## Key Design Decisions

- **Network isolation** — logging / monitoring / tracing / cache each on their
  own bridge network; services only join what they need.
- **Service discovery by name** — everything addressed via Docker's embedded
  DNS (`elasticsearch:9200`), never IPs.
- **Healthcheck gating** — Kibana/Zipkin/Logstash use
  `depends_on: condition: service_healthy` so nothing starts against a cold ES.
- **Read-only config mounts** (`:ro`) — containers can't mutate their own config.
- **Named volumes** (`esdata1`, `grafana-storage`) — data survives full teardown.
- **Resource caps** — ES heap pinned to 512m for a homelab box.

## Lessons Learned (the real value of this repo)

1. **Elasticsearch 9.x enables security + TLS by default.** A healthcheck
   copied from an older tutorial silently failed → `unhealthy` → whole stack
   blocked. Fix for dev: `xpack.security.enabled=false`; for prod: authenticated
   `curl -ku` probe.
2. **Env var renames are invisible bugs** — `ELASTICSEARCH_URL` is ignored in
   Kibana 8+/9+; the working key is `ELASTICSEARCH_HOSTS`.
3. **Zipkin's `ES_HOSTS` needs the REST port (9200)**, not the transport
   port (9300). Timeouts that look like network issues were a wrong port.
4. **Mounting a non-existent host path makes Docker create it — empty.**
   Prometheus then died with `no such file` while compose reported success.
   Always verify files exist before `up`.
5. **`unhealthy` ≠ broken** — Zipkin's API answered fine while its probe
   failed. Query the service directly before trusting the healthcheck.
6. **A valid config without `scrape_configs` is a silent no-op** — Prometheus
   started happily and scraped… nothing. Validate with `promtool check config`.
7. **VMs drift clocks after suspend/resume.** An ~9-minute offset broke
   PromQL time ranges and triggered "server time is out of sync". Fixed with
   `timedatectl set-ntp true` + `open-vm-tools` time sync. Monitoring systems
   are only as good as their timestamps.

## Next Steps

- [ ] Custom metrics exporter from [system-monitor](https://github.com/ParsaZarabi/system-monitor), scraped here
- [ ] Filebeat agents instead of raw TCP input
- [ ] ES snapshot lifecycle + index ILM
- [ ] Harden: ES security on, TLS, secrets via `.env` (gitignored)

---
Built while completing a Docker engineering course — every lesson above was
debugged live, not copy-pasted. 🐳
