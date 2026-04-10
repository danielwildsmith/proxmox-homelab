# Proxmox Homelab — System Overview

Documentation for my single-node Proxmox VE homelab. Covers hardware, container layout, storage, backups, monitoring, and architecture decisions.

---

## Hardware

| Component | Details |
|---|---|
| CPU | Ryzen 5 3600X — 6 cores / 12 threads |
| RAM | 24 GB DDR4 |
| Boot drive | 128 GB SSD |
| Primary data drive | 2 TB IronWolf internal SATA HDD |
| Backup drive | 2 TB USB HDD |
| GPU | NVIDIA GTX 1060 6 GB |

---

## Architecture Principles

- **All LXC containers** — lower overhead than VMs; critical on a single-node setup
- **All unprivileged** — GPU passthrough handled via udev rules and cgroup device entries
- **ext4 directory storage** — no LVM thin pool; disk usage is always visible with `df -h`
- **GPU shared** between Immich (CUDA ML inference) and Jellyfin (NVENC transcoding)
- **Tailscale subnet router** — all services reachable remotely via Tailscale; no port forwarding
- **No RAID** — a 3-2-1 backup strategy covers the real risks (deletion, corruption, disaster)

---

## Service Catalog

| CT ID | Service | Purpose | Access | Key Dependency |
|---|---|---|---|---|
| CT 100 | Tailscale | Subnet router — remote access to all LAN services | Tailscale admin console | — |
| CT 101 | Prometheus + Grafana | Metrics collection, dashboards, alerting | Grafana :3000, Prometheus :9090 | node_exporter on all LXCs |
| CT 102 | Immich | Photo library — upload, browse, ML face/object tagging | :2283 | IronWolf HDD, GPU |
| CT 103 | Jellyfin | Media server — movies and TV with hardware transcoding | :8096 | IronWolf HDD, GPU |
| CT 110 | Proxmox Backup Server | LXC snapshots + restore, DB dump target | :8007 | USB backup HDD |

All containers start automatically on host boot (`onboot: 1`).

---

## Resource Budget

> **Binding constraint: SSD space.** RAM and CPU have comfortable headroom; the 128 GB SSD is the resource to watch as new services are added.

| Container | vCPU | RAM | SSD Disk |
|---|---|---|---|
| Proxmox OS | — | ~1.5 GB | ~30 GB |
| CT 100 — Tailscale | 1 | 256 MB | 8 GB |
| CT 101 — Monitoring | 2 | 2 GB | 16 GB |
| CT 102 — Immich | 4 | 8 GB | 32 GB |
| CT 103 — Jellyfin | 2 | 4 GB | 20 GB |
| CT 110 — PBS | 2 | 2 GB | 16 GB |
| **Total (current)** | **11 / 12** | **~17.75 GB / 24 GB** | **~122 GB / 128 GB (~6 GB free)** |

- **CPU headroom**: 1 thread free
- **RAM headroom**: ~6.25 GB free — planned additions (KOReader sync server, static site) each need <256 MB
- **SSD headroom**: ~6 GB remaining after Prometheus disk expansion — the real ceiling for future LXCs

---

## Storage Layout

```
128 GB SSD  →  /var/lib/vz/
├── Proxmox OS (~30 GB)
├── All LXC root disks
└── Immich PostgreSQL DB (inside CT 102 rootfs)
    └── 2–8 GB typical for 20k–50k photos; 32 GB allocation is safe

2 TB IronWolf SATA HDD  →  /mnt/ironwolf/
├── /mnt/ironwolf/photos     ←  bind-mounted into CT 102 at /mnt/photos
└── /mnt/ironwolf/media      ←  bind-mounted into CT 103 at /mnt/media

2 TB USB HDD  →  /mnt/usbbackup/
├── /mnt/usbbackup/pbs/          ←  PBS datastore (LXC snapshots)
├── /mnt/usbbackup/immich-db/    ←  nightly pg_dump exports
└── /mnt/usbbackup/photos/       ←  rsync mirror of primary photos
```

> The Immich DB stores ML embeddings and metadata only. Actual photo files live on the IronWolf HDD.

---

## Backup Strategy

| Data | Copy 1 (live) | Copy 2 (local) | Copy 3 (offsite) |
|---|---|---|---|
| Photos | IronWolf HDD | USB backup HDD (nightly rsync) | Backblaze B2 (nightly rclone) |
| LXC containers | Running state | PBS snapshots on USB HDD — 7d/4w/2m retention | — |
| Immich DB | Live PostgreSQL | Nightly pg_dump on USB HDD — 14-day rolling | — |
| Media (movies/TV) | IronWolf HDD | Not backed up | — |

> Media is re-downloadable and intentionally excluded from backup.

---

## Monitoring Stack

Prometheus and Grafana run together in CT 101. The following exporters are deployed:

| Exporter | Location | What it covers |
|---|---|---|
| `node_exporter` | Every LXC | CPU, RAM, disk usage, network I/O |
| `nvidia_smi_exporter` | CT 102, CT 103 | GPU utilization, VRAM usage, temperature |
| `cAdvisor` | CT 102 | Docker container metrics (Immich services) |
| `blackbox_exporter` | CT 101 | HTTP uptime probing of all service UIs |

---

## Availability & Single Points of Failure

Single-node homelab — no high availability. All services recover via `onboot: 1` after a host reboot. For a full host failure, recovery is PBS restore to bare metal.

| SPOF | Impact | Mitigation |
|---|---|---|
| IronWolf HDD failure | Photos and media offline | rsync copy on USB HDD + Backblaze B2; SMART health visible via Proxmox (internal SATA) |
| USB backup HDD failure | No new PBS snapshots or pg_dump exports | Live data intact; Grafana alerts on failed backup jobs |
| Proxmox host failure | Everything offline | Rebuild from PBS snapshots; Immich photos survive on IronWolf HDD |
| Tailscale tunnel down | Remote access lost | LAN access unaffected; Tailscale reconnects automatically |
| SSD fills up | LXC writes fail | Monitor with node_exporter; Grafana alerts at >85%; ~6 GB headroom after Prometheus expansion |

---

## Planned Additions

All fit within existing RAM and CPU headroom. SSD impact is small (~2–4 GB each).

| Service | CT | Purpose |
|---|---|---|
| KOReader sync server | CT 104 | Sync reading progress and bookmarks across KOReader devices |
| Static site | CT 105 | Personal site served by nginx |
