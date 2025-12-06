# OCI Cloud-Native Platform (Always Free) — GitOps · k3s · Observability · Backups

Guía práctica y repositorio de referencia para diseñar, desplegar y operar infraestructura en **Oracle Cloud Infrastructure (OCI)** con enfoque en **Always Free**, arquitectura cloud-native y buenas prácticas operativas.

---

## Qué incluye
- **OCI Foundation**: compartments, IAM, VCN/subnets, rutas y NSGs
- **Compute**: Oracle Linux 9 (ARM/AMD según disponibilidad)
- **Kubernetes “lite”**: **k3s** (control-plane + worker)
- **Entrada**: ingress-nginx (Helm)
- **TLS**: cert-manager + Let’s Encrypt
- **Observabilidad**: métricas + logs + dashboards + alertas
- **Backups & DR**: Object Storage + retención + runbooks
- **GitOps**: Terraform + CI + Argo CD/Flux

---

## Estado del proyecto
### Hecho
- ✅ Estructura base en OCI: compartments, VCN, subredes, gateway(s), NSGs y route tables
- ✅ Provisionado de instancias + bootstrap base
- ✅ Cluster k3s operativo (multi-nodo)
- ✅ Ingress NGINX instalado por Helm
- ✅ App demo “whoami” accesible por HTTP a través del Ingress

### En progreso
- 🟡 TLS automático con cert-manager (staging → prod)
- 🟡 Publicación por 80/443 y cierre de NodePorts directos a Internet
- 🟡 Observabilidad: Prometheus/Grafana + Loki
- 🟡 GitOps: Argo CD (auto-sync)
- 🟡 Backups: snapshots + Object Storage + pruebas de restore

---

## Arquitectura (visión)
```text
                 Internet
                    |
               80 / 443
                    |
        +------------------------+
        |        OCI VCN         |
        |   public + private     |
        +------------------------+
           |                |
           | k3s internal   |
           v                v
+-------------------+  +-------------------+
| control-plane     |  | worker(s)         |
| - k3s server      |  | - k3s agent       |
| - ingress-nginx   |  | - apps/db/jobs    |
| - (gitops/obs)    |  | - storage         |
+-------------------+  +-------------------+

Backups -> OCI Object Storage
Observability -> Grafana/Prometheus/Loki (en k8s)
GitOps -> Terraform + CI + Argo CD/Flux
```

---

## Índice
- [Quickstart](#quickstart)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Compartments e IAM](#compartments-e-iam)
- [Networking](#networking-vcn-subnets-nsgs)
- [Storage](#storage-backups--retención)
- [Compute](#compute-always-free)
- [Seguridad](#seguridad--cumplimiento)
- [Observabilidad](#observabilidad)
- [Optimización Always Free](#optimización-always-free)
- [Roadmap GitOps](#roadmap-gitops)
- [Checklist](#checklist)
- [Licencia](#licencia)

---

## Quickstart
### Requisitos previos
- Cuenta OCI (Always Free)
- Acceso a una región
- Terraform (opcional, recomendado)
- GitHub Actions / GitLab CI (opcional)
- Cloud Shell (opcional)

### Comprobaciones rápidas (k3s)
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n ingress-nginx get pods,svc
```

---

## Estructura del repositorio
```text
.
├─ docs/
│  ├─ architecture.md
│  ├─ decisions/            # ADRs (decisiones)
│  └─ runbooks/             # operación y restores
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
   └─ (tu producto)/
```

---

## Compartments e IAM
### Compartments (ejemplo)
| Compartment | Uso |
|---|---|
| `prod` | recursos productivos |
| `lab` | pruebas / sandbox |
| `security` | vault, keys, posture |

### Grupos recomendados
- `administrators`
- `developers`
- `operators`
- `auditors`

### Enfoque de políticas
- Políticas por compartment
- Accesos por rol
- Revisión periódica

---

## Networking (VCN, subnets, NSGs)
### Diseño base
- VCN: CIDR amplio (ej. `10.1.0.0/16`)
- Subnet pública: front/ingress/bastion
- Subnet privada: DB/jobs/servicios internos (opcional)
- IGW para pública, NAT para privada

### NSGs típicas
- `nsg-public-ingress`: 80/443 hacia ingress
- `nsg-ssh-restricted`: 22 para administración
- `nsg-k3s-internal`: tráfico k3s entre nodos
- `nsg-db-storage` (si aplica): puertos DB/MinIO solo internos

---

## Storage (Backups & retención)
### Object Storage
- Buckets privados para backups y artefactos
- Ciclo de vida por tiers (Infrequent / Archive) según retención

### Block Volumes
- Snapshots periódicos
- Restauración documentada (runbook)

---

## Compute (Always Free)
- Oracle Linux 9
- Distribución típica:
  - Control-plane: servicios de plataforma (ingress, gitops/obs)
  - Worker: cargas de trabajo (apps, datos, jobs)

---

## Seguridad & cumplimiento
- Vault/KMS para gestión de secretos y cifrado (si aplica)
- Cloud Guard para postura y alertas
- Segmentación por NSG y mínimo privilegio en IAM
- API server del clúster no expuesto públicamente

---

## Observabilidad
Objetivo:
- métricas + logs centralizados
- dashboards P50/P95/P99
- alertas accionables

Implementación prevista:
- Prometheus + Grafana
- Loki
- OpenTelemetry (cuando aplique)

---

## Optimización Always Free
- Separación `prod` / `lab`
- Apagado programado en `lab`
- Lifecycle en Object Storage
- Selección de shapes ARM/AMD según coste/rendimiento

---

## Roadmap GitOps
- IaC con Terraform
- CI para build/test
- CD declarativo vía Argo CD/Flux
- Runbooks y pruebas de restore como entregables operativos

---

## Checklist
- [ ] IAM: grupos/usuarios + políticas por compartment
- [ ] Networking: VCN + subnets + IGW/NAT + NSGs
- [ ] Storage: buckets + lifecycle + snapshots
- [ ] Compute: instancias + hardening base
- [ ] k3s: clúster multi-nodo operativo
- [ ] ingress-nginx estable
- [ ] TLS con cert-manager
- [ ] Observabilidad (dashboards + alertas)
- [ ] GitOps (Argo CD/Flux)
- [ ] Backups a Object Storage + restore probado

---

## Licencia
MIT License
