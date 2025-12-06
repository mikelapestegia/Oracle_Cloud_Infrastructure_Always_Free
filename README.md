# oci-alwaysfree-k3s-gitops-platform

Cloud-native platform on Oracle Cloud Always Free: k3s, ingress, TLS, observability, GitOps and backups for a production-like portfolio.

---

## About

Reference implementation of a cloud-native platform on Oracle Cloud Infrastructure (Always Free). Includes a multi-node k3s cluster, ingress-nginx, automatic TLS with cert-manager, observability (Prometheus/Grafana/Loki), GitOps (Terraform + Argo CD) and backup strategies using OCI Object Storage.

This repository is also the canonical source of truth for the cluster state (GitOps), the infrastructure as code and the operational runbooks.

## Project status

See `docs/bitacora/2025-12-06-dia1-oci-k3s.md` for the current, detailed state of the cluster and OCI resources (Day 1 log).

High-level status:

- ✅ OCI base infrastructure (compartments, VCN/subnets, NSGs, gateways, IAM, Object Storage)
- ✅ k3s control plane and worker node provisioned on Always Free shapes
- 🔄 Ingress, TLS, observability, GitOps and backups being wired into the cluster

## Repository structure (planned)

```text
oci-alwaysfree-k3s-gitops-platform/
├─ README.md
├─ LICENSE
├─ docs/
│  ├─ architecture.md
│  ├─ decisions/
│  │  └─ adr-0001-oci-k3s-platform.md
│  ├─ runbooks/
│  │  ├─ rb-001-bootstrap-k3s.md
│  │  └─ rb-002-https-cert-manager.md
│  └─ bitacora/
│     └─ 2025-12-06-dia1-oci-k3s.md
├─ infra/
│  ├─ terraform/
│  └─ iam-policies/
├─ bootstrap/
│  ├─ cloud-init/
│  └─ scripts/
├─ k8s/
│  ├─ base/
│  ├─ ingress/
│  ├─ cert-manager/
│  ├─ observability/
│  └─ gitops/
└─ apps/
   ├─ demo-whoami/
   └─ (product apps)
```

> Note: for now only `README.md`, `LICENSE` and `docs/bitacora/2025-12-06-dia1-oci-k3s.md` are required to get the repository in a solid, shareable state.

## Next steps

- [ ] Move existing Terraform, scripts and Kubernetes manifests into `infra/`, `bootstrap/` and `k8s/`
- [ ] Add ADRs under `docs/decisions/`
- [ ] Add runbooks under `docs/runbooks/`
- [ ] Expose a minimal demo app under `apps/` via ingress + TLS

## License

This project is licensed under the MIT License. See `LICENSE` for details.
