# Homelab k3s HA cluster: a complete architecture guide

**A three-node k3s cluster with embedded etcd, Traefik v3, and cert-manager can serve as a production-grade homelab platform** that handles both internal LAN services and internet-facing applications with automatic TLS. This guide covers the full stack: HA control plane with kube-vip, on-demand worker node lifecycle, dual internal/external ingress with Traefik, and proxying to LXC containers outside the cluster. All examples target k3s v1.34.x (etcd 3.6, Kubernetes 1.34), Traefik v3.x (Helm chart v39.x), and cert-manager v1.16.x.

---

## Part 1: k3s HA architecture with embedded etcd

### How the three-node control plane works

K3s implements HA by embedding etcd directly into the k3s binary — each server node runs the API server, controller-manager, scheduler, **and** an etcd member in a single process. This "stacked" topology eliminates external database infrastructure entirely.

The first server bootstraps the etcd cluster with `--cluster-init`:

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=MySecretToken sh -s - server \
    --cluster-init \
    --tls-san=192.168.1.10  # VIP address for kube-vip
```

Subsequent servers join by pointing to any existing server:

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=MySecretToken sh -s - server \
    --server https://192.168.1.11:6443 \
    --tls-san=192.168.1.10
```

Once a node has joined and etcd data exists on disk, the `--cluster-init` and `--server` flags are ignored on subsequent restarts — the node remembers its cluster membership.

**Three nodes is the minimum for HA** because etcd uses the Raft consensus algorithm, which requires a majority (quorum) to commit writes. With **3 nodes, quorum = 2**, meaning the cluster tolerates exactly one node failure. A 4-node cluster still only tolerates one failure (quorum = 3), so odd numbers are always preferred:

| Cluster size | Quorum required | Tolerated failures |
|:---:|:---:|:---:|
| 1 | 1 | 0 |
| 3 | 2 | **1** |
| 5 | 3 | **2** |

When one master goes down, the remaining two maintain quorum — the cluster continues accepting reads and writes normally, and a new etcd leader is elected within seconds. When two masters go down, **quorum is lost**: the API server becomes unavailable for mutations, though existing workloads on nodes continue running undisturbed.

**K3s v1.34.x ships with etcd 3.6** (v3.6.7-k3s1), a significant upgrade from etcd 3.5 in earlier versions. A critical caveat: there is no safe direct upgrade path from etcd 3.5 to 3.6 except through etcd v3.5.26 (included in k3s v1.32/v1.33 January 2026 patches). Skipping this intermediate step can cause "zombie members" — previously removed etcd nodes reappearing and disrupting quorum. Always upgrade through v1.32/v1.33 first.

### Etcd quorum and recovery

Raft leader election works through heartbeats and randomized timeouts. The leader sends periodic heartbeats to followers; if a follower misses heartbeats beyond a randomized election timeout, it promotes itself to candidate and requests votes. The randomization prevents split votes in practice. Slow storage (SD cards, spinning disks) can cause spurious elections — **SSD storage is strongly recommended** for etcd performance.

If quorum is permanently lost, k3s provides a recovery mechanism:

```bash
# On the surviving server, reset etcd to single-member
k3s server --cluster-reset --cluster-reset-restore-path=/path/to/snapshot

# Restart normally
systemctl restart k3s

# On the other servers, wipe etcd and rejoin
systemctl stop k3s
rm -rf /var/lib/rancher/k3s/server/db
systemctl start k3s
```

K3s takes automatic etcd snapshots every 12 hours (configurable via `--etcd-snapshot-schedule-cron`). **Always back up the server token** (`/var/lib/rancher/k3s/server/token`) alongside snapshots — it encrypts the bootstrap data stored in etcd.

### Why k3s masters schedule workloads by default

Unlike standard Kubernetes (kubeadm), k3s server nodes carry **no `NoSchedule` taint** — workloads schedule freely on masters. This is intentional for resource-constrained environments like homelabs where every node counts. In a 3-node cluster where all nodes are masters, this means your applications run alongside the control plane.

This works well for moderate workloads, but protect control plane resources with kubelet reservations:

```yaml
# /etc/rancher/k3s/config.yaml
kubelet-arg:
  - "kube-reserved=cpu=300m,memory=512Mi"
  - "system-reserved=cpu=200m,memory=256Mi"
```

If you later add dedicated workers and want to isolate masters, taint them with `--node-taint CriticalAddonsOnly=true:NoSchedule` at install time or post-hoc via `kubectl taint`.

### kube-vip provides the single entry point

Without kube-vip, agents and `kubectl` must target a specific server IP — if that server dies, connectivity breaks. **kube-vip creates a floating Virtual IP (VIP)** that moves between control plane nodes automatically.

In homelab ARP mode, kube-vip runs as a DaemonSet on control plane nodes. One pod is elected leader via Kubernetes lease objects and binds the VIP to its host interface. If the leader fails, another pod acquires the lease and sends gratuitous ARP to redirect traffic — **failover takes 3–10 seconds**.

Deploy kube-vip by placing this manifest in `/var/lib/rancher/k3s/server/manifests/kube-vip.yaml` on the first server before cluster init:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-vip-ds
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
      containers:
      - name: kube-vip
        image: ghcr.io/kube-vip/kube-vip:v1.0.4
        args: ["manager"]
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_interface
          value: eth0               # Your host NIC
        - name: address
          value: 192.168.1.10       # Your chosen VIP
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: vip_leaderelection
          value: "true"
        - name: vip_leaseduration
          value: "5"
        - name: vip_renewdeadline
          value: "3"
        - name: vip_retryperiod
          value: "1"
        - name: vip_nodename
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          capabilities:
            add: ["NET_ADMIN", "NET_RAW", "SYS_TIME"]
      hostNetwork: true
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
```

After setup, agents join using the VIP: `--server https://192.168.1.10:6443`. K3s agents include a built-in client-side load balancer that, after initial registration via the VIP, discovers all server endpoints and maintains direct connections to each — providing resilience even if the VIP temporarily fails.

Verify cluster health with:

```bash
kubectl get nodes -o wide
# All three should show Ready, roles: control-plane,etcd,master

k3s etcd-snapshot list
# Shows automatic snapshots
```

---

## Part 2: On-demand worker node lifecycle

### Agent auto-join and identity persistence

A k3s agent joins with a single command and **automatically rejoins after every reboot**. The installer creates a `k3s-agent` systemd service enabled at boot. Between restarts, the agent persists its identity via several key files:

| Path | Purpose |
|------|---------|
| `/etc/rancher/node/password` | Node password — matched against a Secret in `kube-system` |
| `/var/lib/rancher/k3s/agent/` | Kubelet certificates and containerd state |
| `/etc/systemd/system/k3s-agent.service.env` | Server URL and token |

When the agent starts, it reconnects using its stored credentials and the node transitions from `NotReady` back to `Ready`. The node retains its original name and identity — Kubernetes sees it as the same node returning, not a new one.

One critical detail: if you delete `/etc/rancher/node/password` without removing the node from the cluster, the agent **cannot rejoin** because the password no longer matches the stored Secret. Fix this by running `kubectl delete node <name>` first, which clears the password Secret.

### What happens when a node disappears

When a node powers off unexpectedly, Kubernetes follows a precise timeline:

**T+0s**: Kubelet stops sending heartbeats (normally every 10s via node lease).

**T+40s** (default `node-monitor-grace-period`): The node controller marks the node's condition as `Unknown` and applies two taints automatically: `node.kubernetes.io/unreachable:NoExecute` and `node.kubernetes.io/not-ready:NoExecute`.

**T+340s** (40s + default **300s toleration**): Pods without matching tolerations are evicted. Kubernetes automatically injects tolerations with `tolerationSeconds=300` for both `not-ready` and `unreachable` on every pod via the `DefaultTolerationSeconds` admission controller.

The old `pod-eviction-timeout` flag is effectively deprecated — **taint-based eviction controls the actual timing now**. The relevant parameters are `default-not-ready-toleration-seconds` and `default-unreachable-toleration-seconds` on the API server.

How pods behave depends on their controller:

- **Deployments/ReplicaSets**: New replicas are created on healthy nodes immediately after eviction. Old pods remain stuck in `Terminating` until the node returns or they're force-deleted.
- **StatefulSets**: Pods are marked `Terminating` but **not replaced** until deletion is confirmed — Kubernetes avoids split-brain by refusing to run two instances simultaneously. This is the most problematic case for unexpected shutdowns.
- **DaemonSets**: Never evicted — they have built-in tolerations with no timeout. They simply wait for the node to return.
- **Bare pods**: Marked `Terminating`, permanently lost (no controller to recreate them).

For StatefulSet pods stuck after an unexpected shutdown, Kubernetes 1.28+ (GA) provides the **non-graceful node shutdown** feature. Apply the `out-of-service` taint to force-delete pods and immediately detach PVs:

```bash
# Only after confirming the node is truly down
kubectl taint nodes <node> node.kubernetes.io/out-of-service=nodeshutdown:NoExecute

# After the node recovers, remove it
kubectl taint nodes <node> node.kubernetes.io/out-of-service=nodeshutdown:NoExecute-
```

### Tuning timeouts for faster failover

The default 5m40s total eviction time is conservative. For a homelab with on-demand nodes, tighten the parameters on the k3s server:

```yaml
# /etc/rancher/k3s/config.yaml (SERVER)
kubelet-arg:
  - "node-status-update-frequency=5s"

kube-controller-manager-arg:
  - "node-monitor-grace-period=20s"

kube-apiserver-arg:
  - "default-not-ready-toleration-seconds=30"
  - "default-unreachable-toleration-seconds=30"
```

This reduces the timeline to: **T+20s** node marked `Unknown` → **T+50s** pods evicted — roughly 6× faster than defaults. Don't go lower than 20s for the grace period, as brief network blips could trigger false detections.

For per-workload control without changing cluster defaults, set toleration timeouts directly on deployments:

```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 10    # Critical apps: fast failover
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 10
```

### Taints and tolerations for on-demand GPU nodes

Taint the on-demand worker at install time so only designated workloads schedule there:

```yaml
# /etc/rancher/k3s/config.yaml (AGENT - on-demand node)
server: "https://192.168.1.10:6443"
token: "<token>"
node-taint:
  - "node-type=on-demand:NoSchedule"
  - "gpu=true:NoSchedule"
node-label:
  - "node-type=on-demand"
  - "hardware=gpu"
```

`NoSchedule` prevents new pods from landing on this node unless they tolerate the taint. Existing pods are unaffected (unlike `NoExecute`, which would evict them). For workloads targeting this node, you need **both** a toleration (permission to schedule) **and** a nodeSelector (requirement to schedule there):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: media-transcoder
spec:
  replicas: 1
  selector:
    matchLabels:
      app: media-transcoder
  template:
    spec:
      tolerations:
        - key: "node-type"
          operator: "Equal"
          value: "on-demand"
          effect: "NoSchedule"
        - key: "gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      nodeSelector:
        hardware: gpu
      containers:
        - name: transcoder
          image: myrepo/media-transcoder:latest
          resources:
            limits:
              nvidia.com/gpu: 1
```

Without the nodeSelector, the toleration merely *permits* scheduling on the GPU node — it doesn't *require* it. The pod could land on any untainted node instead.

### Graceful drain before planned shutdown

Always drain before a planned power-off to let pods migrate cleanly:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \
  --timeout=90s
```

Automate this with a systemd unit that runs on shutdown:

```ini
# /etc/systemd/system/k3s-drain.service
[Unit]
Description=Drain k3s node before shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target
After=network-online.target k3s-agent.service

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/true
ExecStop=/usr/local/bin/k3s kubectl drain $(hostname) \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \
  --timeout=90s
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

Pair this with an uncordon service that runs on startup to make the node schedulable again when it returns.

---

## Part 3: Traefik ingress for internal and external services

### Customizing the bundled Traefik

K3s v1.34 bundles Traefik v3 and deploys it automatically. **Never edit** the auto-generated manifest at `/var/lib/rancher/k3s/server/manifests/traefik.yaml` — k3s overwrites it. Instead, create a `HelmChartConfig`:

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        redirectTo:
          port: websecure          # Global HTTP→HTTPS redirect
      websecure:
        forwardedHeaders:
          trustedIPs:
            - 10.0.0.0/8
            - 192.168.0.0/16
    service:
      spec:
        externalTrafficPolicy: Local   # Preserve real client IPs
    providers:
      kubernetesCRD:
        allowCrossNamespace: true
    additionalArguments:
      - "--api.dashboard=true"
```

The global `redirectTo` at the entrypoint level is the simplest HTTP→HTTPS redirect approach — it applies to all routes automatically, eliminating the need for per-route redirect middleware.

Traefik v3 introduced **breaking changes** from v2. The CRD API group changed from `traefik.containo.us` to `traefik.io`. Rule matchers now use single-value syntax with logical operators: `Host(\`a.com\`) || Host(\`b.com\`)` instead of `Host(\`a.com\`, \`b.com\`)`. `HostRegexp` uses raw Go regex: `HostRegexp(\`^.+\\.example\\.com$\`)`. And `IPWhiteList` was renamed to `IPAllowList`.

### MetalLB gives Traefik a stable IP

K3s's built-in ServiceLB (Klipper) uses node IPs and host ports — functional but inflexible. **MetalLB in L2 mode** provides a single, predictable VIP from a defined pool. Disable ServiceLB first:

```yaml
# /etc/rancher/k3s/config.yaml
disable:
  - servicelb
```

Then deploy MetalLB and configure an IP pool:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

Traefik's LoadBalancer service will automatically receive an IP like `192.168.1.240`. This is the single IP that all DNS records (both internal and external) will point to.

### Single Traefik instance for both internal and external traffic

Running two separate Traefik deployments (one internal, one external) adds complexity without meaningful benefit in most homelabs. **A single instance with an IP allowlist middleware** cleanly separates the two:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: lan-only
  namespace: kube-system
spec:
  ipAllowList:
    sourceRange:
      - 192.168.1.0/24
      - 10.0.0.0/8
```

Internal services apply this middleware — requests from external IPs get **403 Forbidden**. External services omit it. Since `.home.lan` domains are only resolvable via internal DNS (Pi-hole), they're already invisible to the internet; the allowlist is defense in depth.

### Self-signed CA for internal services

Cert-manager creates a proper CA chain: a self-signed root bootstraps a CA issuer that signs leaf certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: homelab-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: homelab-ca
  secretName: homelab-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: homelab-ca-issuer
spec:
  ca:
    secretName: homelab-ca-secret
```

Issue certificates for internal domains (including wildcards):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-wildcard
  namespace: kube-system
spec:
  secretName: internal-wildcard-tls
  issuerRef:
    name: homelab-ca-issuer
    kind: ClusterIssuer
  dnsNames:
    - "*.home.lan"
    - "home.lan"
  duration: 8760h
  renewBefore: 720h
```

To avoid browser warnings, distribute the CA certificate (`ca.crt` from `homelab-ca-secret`) to client devices' trust stores.

Internal IngressRoute example:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.home.lan`)
      kind: Rule
      middlewares:
        - name: lan-only
          namespace: kube-system
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: internal-wildcard-tls
```

### Let's Encrypt with Cloudflare DNS-01 for external services

DNS-01 challenge is ideal for homelabs: it **works behind NAT without port 80**, supports **wildcard certificates**, and validates entirely through DNS TXT records.

Create a Cloudflare API token with `Zone:DNS:Edit` and `Zone:Zone:Read` permissions, then:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "<your-cloudflare-api-token>"
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

A wildcard certificate covers all subdomains at once:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example-com
  namespace: external-services
spec:
  secretName: wildcard-example-com-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
    - "example.com"
    - "*.example.com"
```

---

## Part 4: Proxying to LXC containers outside the cluster

### Service + Endpoints is the correct approach

The user's architecture has Immich (192.168.1.20) and Nextcloud (192.168.1.21) running as LXC containers on the LAN, not in Kubernetes. Traefik inside k3s needs to route to these IPs.

**ExternalName services won't work** — they require DNS hostnames, not IP addresses, and lack port mapping. The reliable approach is a **Service without a selector** paired with a manually created **Endpoints resource**. Traefik treats these identically to pod-backed services:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: external-services
```

### Immich: complete configuration

Immich runs HTTP on port 3001. TLS terminates at Traefik, and plain HTTP flows to the backend:

```yaml
# Service + Endpoints
apiVersion: v1
kind: Service
metadata:
  name: immich
  namespace: external-services
spec:
  ports:
    - name: http
      port: 3001
      targetPort: 3001
---
apiVersion: v1
kind: Endpoints
metadata:
  name: immich                     # Must match Service name exactly
  namespace: external-services
subsets:
  - addresses:
      - ip: 192.168.1.20
    ports:
      - name: http
        port: 3001
---
# IngressRoute
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: immich
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`immich.example.com`)
      kind: Rule
      services:
        - name: immich
          port: 3001
  tls:
    secretName: wildcard-example-com-tls
```

Traffic flow: **Client → HTTPS → Traefik (TLS termination) → HTTP → 192.168.1.20:3001**. No special scheme or ServersTransport configuration is needed because the backend speaks plain HTTP.

### Nextcloud: handling an HTTPS backend

Nextcloud at 192.168.1.21:443 serves HTTPS with a self-signed certificate. Traefik must connect over HTTPS to the backend but skip certificate verification:

```yaml
# Service + Endpoints
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: external-services
spec:
  ports:
    - name: https
      port: 443
      targetPort: 443
---
apiVersion: v1
kind: Endpoints
metadata:
  name: nextcloud
  namespace: external-services
subsets:
  - addresses:
      - ip: 192.168.1.21
    ports:
      - name: https
        port: 443
---
# ServersTransport: skip TLS verification to backend
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: nextcloud-transport
  namespace: external-services
spec:
  insecureSkipVerify: true
---
# Nextcloud-specific headers and CalDAV/CardDAV redirect
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: nextcloud-headers
  namespace: external-services
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    browserXssFilter: true
    contentTypeNosniff: true
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: nextcloud-wellknown
  namespace: external-services
spec:
  redirectRegex:
    permanent: true
    regex: "https://(.*)/.well-known/(?:card|cal)dav"
    replacement: "https://${1}/remote.php/dav"
---
# IngressRoute with HTTPS backend
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nextcloud
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nextcloud.example.com`)
      kind: Rule
      middlewares:
        - name: nextcloud-headers
        - name: nextcloud-wellknown
      services:
        - name: nextcloud
          port: 443
          scheme: https
          serversTransport: nextcloud-transport
  tls:
    secretName: wildcard-example-com-tls
```

The `scheme: https` tells Traefik to use HTTPS when connecting to the backend. The `serversTransport` reference disables certificate verification for the self-signed cert on the Nextcloud LXC.

**Configure Nextcloud's `config.php`** to trust the Traefik proxy:

```php
'trusted_proxies' => array(
    '10.42.0.0/16',      // k3s pod CIDR (Flannel)
    '10.43.0.0/16',      // k3s service CIDR
    '192.168.1.0/24',    // LAN subnet
),
'overwriteprotocol' => 'https',
'overwrite.cli.url' => 'https://nextcloud.example.com',
```

Without `trusted_proxies`, Nextcloud ignores `X-Forwarded-*` headers and displays reverse proxy configuration warnings. The pod CIDR is needed because traffic arrives at Nextcloud from the Traefik pod's IP (in the `10.42.x.x` range), which gets SNATed through the k3s node's LAN IP.

### Why pods can reach LAN IPs

K3s uses Flannel, which routes pod traffic destined for non-pod IPs through the node's default gateway. Since the k3s nodes sit on the same LAN as the LXC containers, **pods can reach 192.168.1.x addresses by default** — no host networking, additional routes, or special configuration required. Outbound pod traffic is source-NATed through the node's LAN IP.

### EndpointSlice as the modern alternative

The Endpoints API still works perfectly but is considered legacy. For future-proofing, use EndpointSlices instead:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: immich-external-1
  namespace: external-services
  labels:
    kubernetes.io/service-name: immich
    endpointslice.kubernetes.io/managed-by: cluster-admins
addressType: IPv4
ports:
  - name: http
    port: 3001
    protocol: TCP
endpoints:
  - addresses:
      - "192.168.1.20"
    conditions:
      ready: true
```

Traefik v3 reads EndpointSlices natively. For a homelab with a handful of external services, either approach works — Endpoints is simpler, EndpointSlices is more correct.

---

## Complete architecture at a glance

The full network path for both traffic types:

```
EXTERNAL:
Internet → Cloudflare DNS → Router (port 443 forward) → MetalLB VIP (192.168.1.240)
  → Traefik (TLS termination, Let's Encrypt cert) → LXC container (192.168.1.20/21)

INTERNAL:
LAN client → Pi-hole DNS (*.home.lan → 192.168.1.240) → MetalLB VIP
  → Traefik (self-signed CA cert, ipAllowList middleware) → Internal k3s services
```

The key components and their roles:

- **kube-vip** (192.168.1.10): Floating VIP for the k3s API server, used by agents and `kubectl`
- **MetalLB** (192.168.1.240): Stable LoadBalancer IP for Traefik's ingress traffic
- **cert-manager**: Two ClusterIssuers — `homelab-ca-issuer` for `.home.lan` self-signed certs, `letsencrypt-production` for `*.example.com` via Cloudflare DNS-01
- **Traefik v3**: Single instance handling both internal (with `ipAllowList`) and external (unrestricted) IngressRoutes
- **Pi-hole/local DNS**: Resolves `.home.lan` to the MetalLB VIP for split-horizon DNS
- **Service + Endpoints**: Maps external LXC container IPs into Kubernetes service discovery for Traefik routing

This architecture replaces Nginx Proxy Manager entirely: Traefik handles all ingress routing, TLS termination, HTTP→HTTPS redirect, header injection, and proxying to both in-cluster services and external LXC containers — all managed declaratively through Kubernetes manifests with automatic certificate renewal.