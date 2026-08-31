# Homepage Dashboard (Tier 1) — Design

**Date:** 2026-08-30
**Status:** Approved
**Sub-project:** B (Dashboard / launcher) of the "homelab interface" effort. Sub-project A
(observability: Prometheus + Grafana + GPU metrics) is deferred to a later cycle.

## Goal

A single LAN-accessible page that shows every service and device as a live up/down tile
(with latency) plus organized launcher links. **Tier 1 = no credentials, no API
integrations** — availability is checked purely with Homepage's `siteMonitor` (HTTP
HEAD→GET). Richer per-service widgets (k8s pods, Proxmox VM load, TrueNAS pools) are Tier 2,
explicitly out of scope here.

## Why Homepage

Chosen over Glance and Homarr because its config is pure YAML (GitOps-native, lives in this
repo), it has first-class live health checks, and — when we reach Tier 2 — native widgets for
Kubernetes, Proxmox, TrueNAS, Ollama, and Prometheus. The orphaned `apps/glance/` ConfigMap
(a config with no Deployment/Service, never wired into `clusters/kustomization.yaml`) is left
in place for now; removing it is a separate cleanup.

## Architecture

Standard Kubernetes app deployment, mirroring the existing `apps/ollama/` layout and MetalLB
`LoadBalancer` convention.

```
apps/homepage/
├── namespace.yaml       # namespace: homepage
├── configmap.yaml       # homepage config: settings/services/bookmarks/widgets (+ empty docker/kubernetes)
├── deployment.yaml      # ghcr.io/gethomepage/homepage (pinned), config mounted read-only
├── service.yaml         # type: LoadBalancer, port 3000 (MetalLB)
└── kustomization.yaml   # ties the above together
```

Plus the **GitOps fix**: add `../apps/homepage` to `clusters/kustomization.yaml` so it is
actually deployed — closing the same gap that left Glance (and Ollama) applied out-of-band.

### Deployment details / known gotchas

- **`HOMEPAGE_ALLOWED_HOSTS`** — required, or Homepage serves a blank "host not allowed" page.
  Set to `*` for Tier 1 (private LAN, no public exposure). Open item: tighten to the specific
  host once the LB IP / DNS name is pinned.
- **Config is mounted read-only** from the ConfigMap at `/app/config`. Homepage writes logs to
  `/app/config/logs`, so an `emptyDir` is mounted at that path to avoid read-only write errors.
- **Image pinned** to a specific tag (not `latest`) for reproducible GitOps.
- **LoadBalancer IP** left to MetalLB auto-assignment for a clean first deploy (no assumption
  about the pool range). Open item: optionally pin a stable IP + DNS name.

### Templated unknowns (`HOMEPAGE_VAR_*`)

Values not yet known are referenced as `{{HOMEPAGE_VAR_*}}` in `services.yaml` and supplied by
env vars in the Deployment with a `REPLACE_ME` placeholder. The dashboard is fully wired; only
the env values need swapping later. See **Open Items** below.

## Dashboard contents (Tier 1)

`siteMonitor` up/down tiles, grouped:

- **Infrastructure:** TrueNAS `192.168.1.86`, K8s API `192.168.1.110:6443`,
  Proxmox host `prox-nix` (`{{HOMEPAGE_VAR_PROXMOX_IP}}`), K8s worker `192.168.1.182`
- **Network:** UniFi `192.168.1.1`, Xfinity router `10.0.0.1`
- **Apps:** Ollama (`{{HOMEPAGE_VAR_OLLAMA_IP}}:11434`)
- **Info widgets (no creds):** datetime, resources glance, search

## Testing / verification

- `kubectl kustomize apps/homepage` renders without error.
- `kubectl kustomize clusters` includes the homepage resources.
- Post-deploy (requires cluster access, not available from this machine): page loads at the
  assigned LB IP:3000, tiles report status.

## Open Items (info to collect)

- [x] **`prox-nix` (Proxmox host) IP** → `192.168.1.183` (set in deployment.yaml)
- [ ] **Ollama LoadBalancer IP** → set `HOMEPAGE_VAR_OLLAMA_IP` (currently `REPLACE_ME`;
      note `apps/ollama/service.yaml` uses the sanitized doc-range `192.0.2.201`)
- [ ] Confirm K8s worker `192.168.1.182` is the intended node to monitor
- [ ] (Optional) Pin a stable MetalLB IP for the dashboard + add a DNS name
- [ ] (Optional) Tighten `HOMEPAGE_ALLOWED_HOSTS` from `*` to the specific host

## Out of scope (Tier 2 / later)

- API-integrated widgets (k8s/Proxmox/TrueNAS/Ollama/Prometheus) + their secrets strategy
- Observability stack (sub-project A)
- Ingress / DNS / Cert Manager foundation
- Removing the orphaned `apps/glance/` config
