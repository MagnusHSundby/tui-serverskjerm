# tui-serverskjerm

Et terminalbasert dashboard (TUI) for overvåkning av et skole-/labmiljø bestående av
Proxmox-noder og UniFi-nettverksutstyr. Prosjektet henter inn metrics via Prometheus
og viser disse direkte i terminalen

## Tech Stack

| Komponent           | Teknologi                                                                            |
| ------------------- | ------------------------------------------------------------------------------------ |
| TUI-rammeverk       | [Bubble Tea v2](https://github.com/charmbracelet/bubbletea) (Go)                     |
| Programmeringsspråk | Go                                                                                   |
| Metrikk-aggregering | [Prometheus](https://prometheus.io/)                                                 |
| Proxmox-eksporter   | [prometheus-pve-exporter](https://github.com/prometheus-pve/prometheus-pve-exporter) |
| UniFi-eksporter     | [UniFi Poller](https://github.com/unpoller/unpoller)                                 |
| Infrastruktur       | Docker / Docker Compose                                                              |

## Arkitektur

Dataflyten går fra de overvåkede systemene (Proxmox-noder og UniFi), via dedikerte
eksportere som oversetter til Prometheus-metrikker, til Prometheus som lagrer og
aggregerer. TUI-en spør Prometheus' HTTP-API og rendrer resultatet i terminalen.

```mermaid
flowchart LR
    subgraph sources["Overvåkede systemer"]
        elev["Proxmox: elev<br/>172.31.0.10"]
        elev2["Proxmox: elev2<br/>172.31.1.11"]
        master["Proxmox: master<br/>172.31.0.9"]
        unifi["UniFi Controller<br/>192.168.1.1"]
    end

    subgraph stack["Docker Compose-stack — nett: monitoring"]
        pve["pve-exporter<br/>:9221"]
        poller["unifi-poller<br/>:9130"]
        prom["Prometheus<br/>:9090 · 30d retention"]
    end

    tui["TUI — Go / Bubble Tea v2<br/>PrometheusClient"]

    elev -->|Proxmox API| pve
    elev2 -->|Proxmox API| pve
    master -->|Proxmox API| pve
    unifi -->|UniFi API| poller

    pve -->|scrape /pve| prom
    poller -->|scrape| prom

    prom -->|PromQL HTTP-query| tui
```

### Metrikker TUI-en spør om

| PromQL-query               | Mappes til                 |
| -------------------------- | -------------------------- |
| `pve_cpu_usage_ratio`      | `NodeMetrics.CPUUsage`     |
| `pve_memory_usage_bytes`   | `NodeMetrics.MemUsage`     |
| `pve_memory_size_bytes`    | `NodeMetrics.MemTotal`     |
| `unifi_clients_wifi_total` | `UnifiMetrics.WifiClients` |

### Datamodell (Go)

```mermaid
erDiagram
    Metrics ||--o{ NodeMetrics : "Nodes[]"
    Metrics ||--|| UnifiMetrics : "Unifi"

    Metrics {
        slice Nodes
        UnifiMetrics Unifi
        time FetchedAt
    }
    NodeMetrics {
        string Instance
        float64 CPUUsage
        float64 MemUsage
        float64 MemTotal
        bool Up
    }
    UnifiMetrics {
        float64 WifiClients
    }
```

Hver `NodeMetrics` identifiseres av Prometheus-labelen `instance`, som settes per
Proxmox-node i `prometheus.yml`.

## Status

### Fungerer

- Go-prosjekt initialisert med Bubble Tea v2
- Grunnleggende TUI-scaffold med tastaturnavigasjon
- Docker Compose-oppsett med datakilde-stack:
  - Prometheus konfigurert med 30 dagers datalagring
  - PVE Exporter koblet mot tre Proxmox-noder (`elev`, `elev2`, `master`)
  - UniFi Poller for nettverksmetrikk

### Gjenstår / planlagt

- [ ] Hente og vise CPU-, RAM- og diskbruk per Proxmox-node
- [ ] Oversikt over kjørende VM-er og containere per node
- [ ] Nettverksstatus fra UniFi (tilkoblede enheter, trafikk, AP-status)
- [ ] Faktisk dashboard-layout i TUI (paneler, farger, sanntidsoppdatering)

## Kjøring

### Forutsetninger

- [Go 1.21+](https://go.dev/dl/)
- [Docker](https://docs.docker.com/get-docker/) og Docker Compose

### Start datakilde-stacken

```bash
cp pve-exporter/pve-exporter.yml.example pve-exporter/pve-exporter.yml
# Fyll inn kredentialer i pve-exporter.yml

docker compose up -d
```
