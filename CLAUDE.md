# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Ansible playbooks that provision a k3s Kubernetes cluster on Raspberry Pi 5 nodes (`pi1` master, `pi2`–`pi5` workers on `192.168.4.0/24`) and install the cluster's add-ons and workloads. There is no application code: everything is Ansible YAML plus raw Kubernetes manifests.

## Commands

```shell
# Full provisioning (OS prep, k3s, then all add-ons)
ansible-playbook main.yaml

# Re-run a single add-on via its tag: helm, cert-manager, metallb, longhorn, monitoring, pihole, torrent, jellyfin
ansible-playbook main.yaml --tags pihole

# apt dist-upgrade + reboot all nodes
ansible-playbook os-update.yaml

# Lint (same check CI runs on every push/PR via ansible/ansible-lint@v6)
ansible-lint

# Fetch kubeconfig for local kubectl use
scp pi1:/etc/rancher/k3s/k3s.yaml k3s.yaml
sed -i 's/127.0.0.1/192.168.4.10/g' k3s.yaml
```

`ansible.cfg` points at `inventory.ini` automatically. Control machine needs the collections in `requirements.yml` (`kubernetes.core`, `community.general` — install with `ansible-galaxy collection install -r requirements.yml`), and the 1Password CLI (`op`) signed in: `playbooks/cert-manager.yaml` reads the Cloudflare API token with the `community.general.onepassword` lookup. CI's `ansible-lint` run also installs from `requirements.yml` (passed via the action's `requirements_file` input) — without it, lint can't resolve any `kubernetes.core.*` module and fails every add-on playbook.

## Architecture

- **`main.yaml`** runs everything in order: cluster-wide node prep (cgroups in `/boot/firmware/cmdline.txt`, `open-iscsi` for Longhorn), k3s server on `master` (with `servicelb` disabled so MetalLB handles LoadBalancers), k3s agents on `workers`, then each `playbooks/*.yaml` through `import_playbook` with a tag. Order matters: Helm → cert-manager → MetalLB → Longhorn → monitoring → PiHole → qbittorrent → Jellyfin, because later workloads depend on the `longhorn` StorageClass, the `letsencrypt-cloudflare` ClusterIssuer and MetalLB IPs. `monitoring` (kube-prometheus-stack) runs right after Longhorn since its Prometheus/Alertmanager/Grafana PVCs need the `longhorn` StorageClass and its Grafana ingress needs the ClusterIssuer.
- **Workers join through cross-play facts.** The worker play reads the join token from `hostvars[groups['master'][0]]['k3s_token']` (set in the master play) and the master IP from `hostvars['pi1']['ansible_default_ipv4']` (gathered in the first play). Running the worker play on its own, or skipping fact gathering for `pi1`, breaks the join.
- **Add-on playbooks** (`playbooks/`) all target `master`, set `K8S_AUTH_KUBECONFIG=/etc/rancher/k3s/k3s.yaml` and use `kubernetes.core.helm` / `kubernetes.core.k8s` on the Pi itself, so nothing runs from a local kubeconfig. They load `../config.yaml` (the path is relative to the playbook). Raw manifests in `playbooks/manifests/` are applied with `lookup('file', ...) | from_yaml` (single doc) or `from_yaml_all` (multi-doc).
- **`config.yaml`** is the single place for pinned versions (k3s and Helm charts). Version bumps go here; container image tags for qbittorrent and `vpn-killswitch` live in `playbooks/manifests/qbittorrent.yaml`.
- **Ingress/TLS:** k3s's bundled Traefik is the ingress controller, exposed on `192.168.4.201`. Services use `*.picakelab.net` hosts with the `cert-manager.io/cluster-issuer: letsencrypt-cloudflare` annotation (Let's Encrypt DNS-01 through Cloudflare).
- **IP allocation:** MetalLB pool is `192.168.4.201-254` (L2). `.201` is Traefik, `.202` is PiHole (web and DNS services share it via `metallb.universe.tf/allow-shared-ip`). When you add a LoadBalancer service, pick an unused address in that pool.
- **Jellyfin** is not run on the cluster: it lives on the Synology NAS at `192.168.1.180:8096`, and `playbooks/manifests/jellyfin.yaml` only proxies it. The Service has no selector and points at the NAS through a hand-written EndpointSlice, behind a Traefik Ingress on `jellyfin.picakelab.net`. If the NAS IP changes, update the EndpointSlice.
- **qbittorrent** runs alongside a `vpn-killswitch` sidecar. The qbittorrent container's liveness and readiness probes hit the sidecar's `/killswitch` endpoint on port 9000, so the pod goes unready or restarts when the VPN (checked by ASN) drops.

## Gotchas

- Node IPs are duplicated across `inventory.ini`, the PiHole `customDnsEntries` and the kubeconfig instructions.
