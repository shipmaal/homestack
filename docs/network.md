# Homelab Network Architecture – Conceptual Blueprint (rev 2025‑04‑29)

This page captures **the destination** rather than the exact CLI journey.  It should read like a highway map: which lanes exist, what travels on each, and where the guard‑rails sit.  Implementation snippets live elsewhere in Git.  

---

## 1  Bird’s‑eye View

```
                       +-------------------- WAN ------------------+
                       |                                           |
            (Public IP / CF Tunnel)                                |
                       |                                           |
                       v                                           |
                 ┌──────────────┐           Metrics / Logs         |
   Internet ---> |  DMZ  (150)  |-----------------------+          |
                 | Edge Ingress |                       |          |
                 └──────┬───────┘                       |          |
                        | mTLS / HTTPS                  v          |
                 ┌──────▼───────┐              ┌────────────────┐  |
                 | App‑Prod     |  r/w data    | Infra‑Prod     |--+
                 | (120 + 121+) |------------->|  Core (20)     |
                 └──────────────┘              └────────────────┘
                              ^                        ^
                              |                        |
                              |                        | SSH / Admin
                 ┌────────────▼────────────────────────┴───────────┐
                 |                Management  (10)                 |
                 |  NetBox • Vault • Grafana • Flux • Ansible      |
                 └─────────────────────────────────────────────────┘
```

* **DMZ (150)** – first stop from the Internet; terminates TLS, runs WAF, or tunnels.  Serves *only* edge components (MetalLB, Ingress‑NGINX, cloudflared, etc.).
* **App‑Prod (120 … 129)** – production application clusters and their data‑stores.  No direct WAN reachability.
* **Infra‑Prod (20)** – OpenStack control‑plane, Ceph RGW, monitoring stack.  Provides the platform but never shows a customer‑facing port.
* **Mgmt (10)** – Underlay network for phyiscal access.
* **Storage (30 / 31)** – Ceph public & cluster networks; jumbo frames; isolated at L2.
* **Home / Client (172.16.0.0/20)** – Wi‑Fi, IoT, guest devices; purely consumer traffic, tightly firewalled.

---

## 2  IP & VLAN Allocation

| Role / Zone | VLAN | IPv4 Block | Rationale |
|-------------|------|-----------|-----------|
| **Management** | 10 | 192.168.10.0/24 | Small, easy to remember, rarely grows. |
| **Infra‑Prod (platform)** | 20 | 10.0.20.0/24 | Holds control‑plane pods & services only. |
| **Storage public / cluster** | 30 / 31 | 192.168.30.0/24 • 192.168.31.0/24 | Jumbo frames, never routed off hosts. |
| **App‑Prod default** | 120 | 10.0.120.0/24 | First tenant production slice. |
| **Extra tenant slices** | 121‑129 | 10.0.12X.0/24 | Predesigned for future teams/apps. |
| **DMZ / Edge** | 150 | 10.0.150.0/24 | Single ingress zone – easier ACLs & monitoring. |
| **Home Wi‑Fi / IoT / Guest** | 16 / 17 | 172.16.0.0/24 • 172.16.2.0/24 | Separates consumer traffic from lab. |
| **Overlay pods** | – | 10.8.0.0/13 | /16 per k8s cluster, no switch config needed. |

*IPv6*: obtain /56 from ISP (or HE) ⇒ slice /64 per VLAN; ULA `fd75:78a4:b600::/48` for internal‑only addresses.

---

## 3  Guiding Principles

1. **Layered Defence** – traffic crosses at most **one trust boundary** (WAN→DMZ, DMZ→App, App→Infra).  
2. **Least‑privilege Firewalling** – default‑drop on CCR; allowlists reference address‑lists “dmz”, “app”, “infra” rather than raw CIDRs.  
3. **GitOps Source‑of‑Truth** – VLAN trunks, CCR rules, Neutron nets, MetalLB pools, and k8s manifests live in the `homestack` repo and are applied by Flux & Ansible.  
4. **Immutable Platform, Mutable Apps** – Infra‑Prod is cattle; App‑Prod and tenants deploy via CI/CD; DMZ contains no state.  
5. **Zero‑trust Option** – switch NAT → Cloudflare Tunnel without touching App‑Prod or Infra‑Prod.  

---

## 4  Typical North‑South Flow

1. **DNS / TLS** – `api.lab204.dev` resolves to a public IP (or CF tunnel hostname).  
2. **Edge termination** – TLS ends on DMZ ingress controller.  
3. **mTLS hop** – DMZ forwards to NodePort / Istio gateway inside App‑Prod using mutual TLS.  
4. **Data‑store call** – App‑Prod pod reaches Infra‑Prod platform service (e.g., Ceph‐RGW) over cluster network.  
5. **Observability** – Edge and App clusters push metrics & logs northbound to Grafana/Loki living on Infra‑Prod.

---

## 5  Cluster Personas

| Cluster | Runs on | Primary Job | Exposure |
|---------|---------|-------------|----------|
| **Infra‑Prod** | OpenStack VMs + bare‑metal | OpenStack services, Ceph RGW, monitoring, Octavia control | Internal only. |
| **App‑Prod** | Tenant VM or bare‑metal pool on VLAN 120 | Business workloads, DBs | Consumed via DMZ LB. |
| **Edge (DMZ)** | Lightweight k3s on VLAN 150 | Ingress, L4/L7 LB, optional WAF, tunnels | First public touch point. |

Each new tenant gets a **/24 provider VLAN (121‑129)**, a /16 pod CIDR from 10.8.0.0/13, and a /20 service CIDR from 10.96.0.0/12.

---

## 6  Publishing a Service – Playbook

> _Goal_: Expose `GET /orders` from App‑Prod to the Internet while keeping DB and internal APIs private.

1. **Reserve a DMZ VIP** (10.0.150.50) in MetalLB pool via Git commit.
2. **Create Ingress** in Edge cluster mapping `orders.lab204.dev` → backend `orders-api` in App‑Prod.
3. **Add CCR address‑list rule** allowing DMZ → App‑Prod NodePort 32443 (Flux deploys).  
4. **Update DNS** via External‑DNS to add A/AAAA for `orders.lab204.dev`.
5. _Result_: traffic path is **WAN → 150 → 120**, never touching Infra‑Prod or Mgmt.

---

## 7  Observability & Backup

* **Metrics** – Prometheus federated to central stack on VLAN 20.  
* **Logs** – Loki; edge agents push over HTTPS.  
* **Back‑ups** – Velero snapshots → Ceph RGW → nightly rclone to S3 Glacier.  
* **Dashboards** – Grafana on Mgmt; read‑only link proxied through Cloudflare Access.

---

## 8  Growth Roadmap

| Phase | Upgrade | Impact |
|-------|---------|--------|
| **1** | CRS310 (2.5G edge) & CCR2004 router | 10 Gb backbone, VRRP fail‑over with hEX. |
| **2** | CRS504 (100 Gb core) | Core switch upgrade without re‑IP; CRS326 becomes 10 Gb TOR. |
| **3** | Additional App tenant nets (121‑129) | Plug‑n‑play: trunk already present; just add Neutron net + cluster. |

---

_End of conceptual blueprint – see repo dirs `switch/`, `router/`, `openstack/`, `k8s/` for implementation specifics._

