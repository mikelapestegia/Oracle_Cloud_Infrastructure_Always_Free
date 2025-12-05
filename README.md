

[README.md](https://github.com/user-attachments/files/23953450/README.md)
# Oracle Cloud Infrastructure — Always Free (Guía GitOps)

Microservicios con Docker, red/NSG/IAM, seguridad (Vault / Cloud Guard), observabilidad y optimización de costes. Base para IaC (Terraform) y CI/CD.  
🌩️ Guía práctica y referencia para diseñar, desplegar y operar infraestructura en Oracle Cloud Infrastructure (OCI) — enfoque en Always Free Tier (eu-frankfurt-1) y arquitectura de microservicios sobre Oracle Linux 9.

---

Índice
- [Resumen](#resumen)
- [Quiénes y objetivo](#quienes-y-objetivo)
- [Quickstart / Requisitos previos](#quickstart--requisitos-previos)
- [Arquitectura global (visión)](#arquitectura-global-visión)
- [Compartments y IAM (mejores prácticas)](#compartments-y-iam-mejores-prácticas)
- [Redes (VCN, subnets, NSGs)](#redes-vcn-subnets-nsgs)
- [Almacenamiento](#almacenamiento)
- [Compute e instancias (Always Free)](#compute-e-instancias-always-free)
- [Bases de datos](#bases-de-datos)
- [Seguridad y cumplimiento](#seguridad-y-cumplimiento)
- [Observabilidad (monitoring, logging, tracing)](#observabilidad-monitoring-logging-tracing)
- [Optimización de costes y límites Always Free](#optimización-de-costes-y-límites-always-free)
- [Estado del proyecto y roadmap GitOps](#estado-del-proyecto-y-roadmap-gitops)
- [Checklist de implementación](#checklist-de-implementación)
- [Recursos y contacto](#recursos-y-contacto)

---

## Resumen
OCI es una plataforma empresarial con servicios de computación, almacenamiento, redes y seguridad.  
Esta guía combina: diseño operativo, seguridad, optimización de costes y una base para GitOps (IaC + CI/CD) orientada a Always Free Tier en Frankfurt.

---

## Quiénes y objetivo
Objetivo:
- Desplegar una infraestructura segura, escalable y coste-eficiente.
- Soportar microservicios con Docker (evolución a OKE / Kubernetes).
- Tener base para GitOps: repos, pipelines y automatizaciones.

Audiencia:
- DevOps / SRE
- Arquitectos cloud
- Equipos de desarrollo que requieren una base reproducible

---

## Quickstart / Requisitos previos
- Cuenta OCI con acceso a eu-frankfurt-1
- CLI OCI configurado (~/.oci/oci_api_key.pem)
- Usuario con permisos para crear compartments, VCN, instancias y claves KMS
- Opcional: Terraform, GitHub/GitLab CI, Cloud Shell

Ejemplo de configuración OCI CLI (~/.oci/config):
```ini
[DEFAULT]
user=ocid1.user.oc1..xxxxx
fingerprint=ab:cd:ef:12:34:56:78:90
key_file=~/.oci/oci_api_key.pem
tenancy=ocid1.tenancy.oc1..xxxxx
region=eu-frankfurt-1
```

---

## Arquitectura global (visión)
Diagrama conceptual (resumen):
```
VCN: vcn-prod-frankfurt (10.1.0.0/16)
├── Public Subnet (10.1.1.0/24)   -> Apps, Proxy, Bastion, Monitoring (IGW)
└── Private Subnet (10.1.2.0/24)  -> DB, Jobs, servicios internos (NAT/Private)
```

Componentes clave:
- Internet Gateway (IGW) para subnets públicas
- NAT Gateway para salida desde privadas sin IP pública
- NSGs para micro-segmentación
- Object Storage para backups y logs
- Vault / KMS para secretos y cifrado

---

## Compartments y IAM (mejores prácticas)
Compartments (ejemplo)
| Compartment | Uso |
|-------------|-----|
| prod        | Recursos productivos |
| lab         | Desarrollo y pruebas |
| security    | Vault, KMS, Cloud Guard |

Grupos recomendados:
- administrators (acceso full)
- developers (recursos dev)
- operators (operaciones / DBA)
- auditors (solo lectura)

Ejemplos de políticas:
```text
# Administradores
Allow group administrators to manage all-resources in tenancy

# Developers en dev-compartment
Allow group developers to manage instance-family in compartment dev-compartment
Allow group developers to manage virtual-network-family in compartment dev-compartment
Allow group developers to read audit-events in tenancy

# Auditores
Allow group auditors to read all-resources in tenancy
```

MFA y rotación:
- Habilitar MFA (app autenticadora).
- Rotación de API keys: cada 90 días.

---

## Redes (VCN, subnets, NSGs)
VCN:
- Nombre: vcn-prod-frankfurt
- CIDR: 10.1.0.0/16
- DNS: habilitado

Subnets:
- prod-public (10.1.1.0/24): frontends, bastion, proxy
- prod-private (10.1.2.0/24): DB, servicios internos

NSGs recomendadas:
- nsg-proxy: HTTP/HTTPS -> TCP 80/443 from 0.0.0.0/0
- nsg-ssh: SSH restringido -> TCP 22 from <admin-ip>/32
- nsg-internal: comunicación interna -> allow VCN range

Route tables:
- Pública: 0.0.0.0/0 → IGW
- Privada: 0.0.0.0/0 → NAT Gateway (si necesita salida)

Acceso remoto seguro:
- Bastion + WireGuard en instancia dedicada para administración de subred privada.

---

## Almacenamiento
Object Storage:
- Buckets privados para backups, logs y artifacts.
- Ciclo de vida recomendado:
  - 30d → Infrequent Access
  - 90d → Archive
  - 365d → Eliminar (según compliance)
- Encriptación: Customer Managed Keys (KMS)

Block Volumes:
- Tipos: Basic / Balanced / High Performance
- Snapshots diarios, retención: 30 días (ejemplo)

File Storage:
- Uso para directorios compartidos (NFS) entre instancias.

---

## Compute e instancias (Always Free)
Instancias recomendadas (ejemplo):
- Instance 1 — App + Proxy (VM.Standard.A1.Flex) — 2 OCPU / 12 GB — Public
- Instance 2 — DB + Jobs (VM.Standard.A1.Flex) — 2 OCPU / 12 GB — Private
- Instance 3 — Bastion + WireGuard (VM.Standard.E2.1.Micro) — Public
- Instance 4 — Monitoring / Jumpbox (VM.Standard.E2.1.Micro)

Imagen: Oracle Linux 9 (ARM64 donde aplique). Cloud-init para bootstrap de Docker, Nginx, WireGuard.

Evolución:
- Migrar a OKE + node pools para cargas contenizadas y GitOps.

---

## Bases de datos
Opciones:
- Oracle DB (DBCS) — Enterprise con Data Guard (si es necesario)
- MySQL Database Service — managed, HA
- PostgreSQL (managed o en compute) — recomendado para microservicios

Buenas prácticas:
- Aislar en subnets privadas
- Backups automáticos a Object Storage
- Cifrado en reposo con KMS

---

## Seguridad y cumplimiento
Vault & KMS:
- Guardar secretos (DB passwords, API keys, certificados)
- Key rotation y acceso controlado

WAF / DDoS:
- WAF con reglas OWASP, rate-limiting y protecciones personalizadas (/admin/*)
- Monitorizar y escalar protección si la aplicación lo requiere

Audit & Cloud Guard:
- Habilitar logging de auditoría
- Cloud Guard para detección y acciones automáticas (bloqueos, alertas)

Principio:
- Principio de mínimo privilegio en IAM y NSGs

---

## Observabilidad (monitoring, logging, tracing)
Monitoring:
- Métricas: CPU, memoria, disco, latencia, errores
- Dashboards P50/P95/P99

Logging:
- Centralizar logs (instancias, LB, WAF) → Object Storage o servicio de logs
- Integración con Grafana + Loki

Tracing:
- Instrumentar con OpenTelemetry (Java / Python / Node.js)
- Visualizar spans para depuración de latencias

Ejemplo de alertas:
- CPU > 80% (5m)
- Error rate > 1% (5m)
- Storage > 85%

---

## Optimización de costes y Always Free
Always Free ejemplos:
- ARM Ampere A1: hasta 4 OCPUs / 24 GB RAM (combinado)
- Micro VMs: VM.Standard.E2.1.Micro (x2)
- Block Storage: 200 GB
- Load Balancer: 1

Estrategias:
- Apagado programado de entornos no productivos
- Tiering en Object Storage
- Reservas para cargas estables
- Selección de shapes (ARM vs AMD) para coste/rendimiento

Límites y solicitudes:
- Revisar cuotas por región; solicitar aumento si es necesario.

---

## Estado del proyecto y roadmap GitOps
Estado actual (resumen):
- ✅ Suscripción región Frankfurt
- ✅ Compartments: prod, lab, security
- ✅ VCN + subnets + IGW
- ✅ NSGs y route tables configuradas
- 🟡 Solicitud de aumento de cores A1 (CAM-252231) — pendiente

Pendiente / Roadmap:
- Despliegue de instancias (1..4)
- Cloud-init y scripts de bootstrap
- Instalación Docker / docker-compose
- Setups: PostgreSQL/MySQL, n8n, Nginx/Caddy, WireGuard
- Backups automáticos y políticas de retención
- Pipelines GitOps (Terraform + CI/CD)

---

## Checklist de implementación (rápido)
- [ ] Crear tenencia y habilitar regiones
- [ ] Configurar zona horaria y moneda
- [ ] Crear grupos IAM y usuarios; habilitar MFA
- [ ] Crear VCN y subnets (public/private)
- [ ] Configurar IGW / NAT / NSGs / Route tables
- [ ] Crear buckets Object Storage y reglas de lifecycle
- [ ] Desplegar instancias y cloud-init
- [ ] Instalar Docker, configurar stacks de microservicios
- [ ] Configurar Vault / KMS, Cloud Guard y WAF
- [ ] Implementar monitoring, logs y tracing
- [ ] Automatizar con Terraform + CI/CD (GitOps)

---

## Recursos útiles
- Documentación OCI: https://docs.oracle.com/en-us/iaas/
- Console OCI: https://cloud.oracle.com
- Estado de servicios: https://status.oraclecloud.com
- Comunidad: Oracle Cloud Community Forums

---

Autor / Contacto
- Mikel Apestegia — mikelapestegia@gmail.com
- Tenancy OCID: ocid1.tenancy.oc1..aaaaaaaam5ulr6q4pztpx3beybvoykiilojvmhjccn55rtbazrzoo4mdw7xq

Última actualización: 2025-12-05 02:00 CET  
Estado de infraestructura: 🟡 Awaiting A1 limit approval — Región primaria: 🇩🇪 eu-frankfurt-1

---
Notas finales
- Este README es una referencia operativa y guía de despliegue. Si quieres, puedo:
  1) abrir un commit en el repo con este README.md, o
  2) generar una rama/PR con cambios separados (recomendado para revisión).
