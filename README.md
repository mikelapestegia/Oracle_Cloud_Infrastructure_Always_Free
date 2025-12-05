# Oracle_Cloud_Infrastructure_Always_Free
Microservicios con Docker, red/NSG/IAM, seguridad (Vault/Cloud Guard), observabilidad y optimización de costes, con base para IaC (Terraform) y CI/CD.
🌩️ Guía Completa de Configuración Oracle Cloud + GitOps Infrastructure (OCI)
Referencia completa para diseñar, desplegar y operar infraestructura en Oracle Cloud Infrastructure (OCI), incluyendo Always Free Tier en Frankfurt, arquitectura de microservicios con Docker y configuración detallada de seguridad, redes, bases de datos, monitoreo y optimización de costes.

📋 Tabla de Contenidos
Introducción
Arquitectura Global y Tenencia
Gestión de Identidad y Acceso (IAM)
Arquitectura y Configuración de Redes
Almacenamiento
Computación e Instancias
Bases de Datos
Seguridad y Cumplimiento
Monitoreo, Logging y Tracing
Optimización de Costos y Always Free Tier
Estado del Proyecto y Roadmap GitOps
Checklist de Implementación
Recursos, Notas y Autor

📖 Introducción
Oracle Cloud Infrastructure (OCI) es una plataforma de nube empresarial que ofrece servicios de computación, almacenamiento, base de datos, redes y seguridad.
Este documento combina:
Una guía de configuración general de OCI, muy detallada y orientada a entornos empresariales.
Un README específico de infraestructura GitOps sobre Always Free Tier en Frankfurt (eu-frankfurt-1) para microservicios con Docker y Oracle Linux 9.
🎯 ¿Qué lograrás?
✅ Entorno de nube seguro, escalable y bien segmentado.
✅ Gestión centralizada de identidades y accesos (IAM) con buenas prácticas.
✅ Infraestructura de redes, compute, almacenamiento y bases de datos lista para producción.
✅ Arquitectura concreta en Frankfurt usando Always Free Tier (A1, E2.1.Micro).
✅ Seguridad reforzada con WAF, Vault, KMS, Cloud Guard, MFA y NSG.
✅ Observabilidad (monitoring, logging, tracing distribuido).
✅ Optimización de costes, incluyendo reservas y tiering de almacenamiento.
✅ Base sólida para un flujo GitOps con CI/CD y despliegues automatizados.

🏛 Arquitectura Global y Tenencia
1.1 Tenencia y Regiones
Tenancy: Entorno aislado donde residen todos los recursos.
Configuraciones clave de tenencia:
Nombre de Tenencia: Identificador único de tu cuenta (branding y organización).
Región Inicial: Determina latencia y cumplimiento normativo.
Moneda de Facturación: USD, EUR, etc. para informes financieros.
Zona Horaria: Para sincronización de logs y auditoría.
Regiones usadas en esta arquitectura:
Región principal de arquitectura GitOps:
eu-frankfurt-1 (Germany Central - Frankfurt)
Ejemplos adicionales para HA / DR:
eu-madrid-1 (España)
eu-frankfurt-1 (Alemania)
us-phoenix-1, us-ashburn-1, etc. (ejemplos globales).
Resultado esperado:
Tenencia configurada con políticas de seguridad por defecto y regiones necesarias habilitadas.

1.2 Compartments
Organizan lógicamente los recursos:
Compartment
Descripción
Uso
prod
Production environment
Infraestructura productiva
lab
Development/Testing
Desarrollo y pruebas
security
Seguridad y claves
Vault, KMS, Cloud Guard


👥 Gestión de Identidad y Acceso (IAM)
2.1 Estructura Organizativa y Grupos
Control granular sobre permisos de usuarios, grupos y servicios.
Grupos recomendados:
administrators – Acceso total a la tenencia.
developers – Gestión de recursos en compartimentos de dev/lab.
operators – Operaciones diarias (DBA, sysops).
auditors – Solo lectura y auditoría.
Estructura ejemplo:
text
tenencia/
├── administrators
│   ├── Admin Principal
│   └── Admin Secundario
├── developers
│   ├── Backend Developers
│   ├── Frontend Developers
│   └── DevOps Engineers
├── operators
│   ├── Database Administrators
│   └── System Operators
└── auditors
    ├── Security Auditors
    └── Compliance Officers


2.2 Políticas (Policies) de Acceso
Definen qué acciones puede hacer cada grupo en qué recursos.
Ejemplos clave:
Administradores:
text
Allow group administrators to manage all-resources in tenancy
Allow group administrators to use cloud-shell in tenancy
Allow group administrators to read audit-events in tenancy

Desarrolladores (ej. en dev-compartment):
text
Allow group developers to manage instance-family in compartment dev-compartment
Allow group developers to manage virtual-network-family in compartment dev-compartment
Allow group developers to manage database-family in compartment dev-compartment
Allow group developers to read audit-events in tenancy
Allow group developers to use cloud-shell in tenancy

Auditores (solo lectura):
text
Allow group auditors to read all-resources in tenancy
Allow group auditors to read audit-events in tenancy
Allow group auditors to read instances in tenancy


2.3 Usuarios, MFA y API Keys
Usuarios:
Datos básicos: nombre, email corporativo, descripción (rol/departamento).
Asignación a grupos según función (admin, dev, auditor, etc.).
MFA (Multi-Factor Authentication):
Recomendado:
App autenticadora (Google Authenticator/Authy) + backup por SMS.
Beneficio:
Protección contra robo de credenciales.
API Keys y archivo de configuración OCI:
text
[DEFAULT]
user=ocid1.user.oc1..xxxxx
fingerprint=ab:cd:ef:12:34:56:78:90:ab:cd:ef:12:34:56:78:90
key_file=~/.oci/oci_api_key.pem
tenancy=ocid1.tenancy.oc1..xxxxx
region=eu-frankfurt-1

Rotación recomendada: cada 90 días (generar nueva key, actualizar apps, eliminar la antigua).

🌐 Arquitectura y Configuración de Redes
3.1 Vista General (Frankfurt, Always Free)
Arquitectura distribuida en OCI (Frankfurt) con segregación de redes públicas y privadas.
text
┌─────────────────────────────────────────────────────────────┐
│ VCN: vcn-prod-frankfurt (10.1.0.0/16)                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌──────────────────────┐   ┌──────────────────────────┐     │
│ │ Public Subnet        │   │ Private Subnet           │     │
│ │ 10.1.1.0/24          │   │ 10.1.2.0/24              │     │
│ ├──────────────────────┤   ├──────────────────────────┤     │
│ │ - A1 Instance 1      │   │ - A1 Instance 2          │     │
│ │   (Apps + Proxy)     │   │   (Database + Jobs)      │     │
│ │   2 OCPU / 12 GB     │   │   2 OCPU / 12 GB         │     │
│ │                      │   │   No Public IP           │     │
│ │ - Micro Instance 1   │   │                          │     │
│ │   (Bastion+WireGuard)│   │                          │     │
│ │   1 OCPU / 1 GB      │   │                          │     │
│ │                      │   │                          │     │
│ │ - Micro Instance 2   │   │                          │     │
│ │   (Monitor/Jumpbox)  │   │                          │     │
│ │   1 OCPU / 1 GB      │   │                          │     │
│ └──────────────────────┘   └──────────────────────────┘     │
│                                                             │
│                         ▼                                   │
│        Internet Gateway: igw-prod-frankfurt                 │
└─────────────────────────────────────────────────────────────┘


3.2 Virtual Cloud Network (VCN)
Nombre: vcn-prod-frankfurt
CIDR Block: 10.1.0.0/16
Región: eu-frankfurt-1
DNS: habilitado
Internet Gateway: igw-prod-frankfurt
Ejemplo general de VCN por capas:
text
VCN: 10.0.0.0/16
├── Public Subnet (Web Tier)
│   ├── CIDR: 10.0.1.0/24
│   ├── Route: 0.0.0.0/0 → Internet Gateway
│   └── Security List: Permite puertos 80, 443
├── Private Subnet (App Tier)
│   ├── CIDR: 10.0.2.0/24
│   ├── Route: 0.0.0.0/0 → NAT Gateway
│   └── Security List: Permite comunicación VCN
└── Private Subnet (Database Tier)
    ├── CIDR: 10.0.3.0/24
    ├── Route: Local
    └── Security List: Solo comunicación App Tier


3.3 Subnets
Public Subnet
Nombre: prod-public
CIDR: 10.1.1.0/24
Tipo: Public
Route Table: 0.0.0.0/0 → igw-prod-frankfurt
Seguridad: NSG-enabled
Uso:
Frontend services
Bastion hosts
Proxy / reverse proxy
Monitor/Jumpbox
Private Subnet
Nombre: prod-private
CIDR: 10.1.2.0/24
Tipo: Private
Route Table: Default privada (sin IGW; opcional NAT GW)
Seguridad: NSG-enabled
Uso:
Backend services
Bases de datos
Jobs internos (cron, backups, n8n)

3.4 Route Tables
Default Route Table (pública):
text
Destination: 0.0.0.0/0
Target: igw-prod-frankfurt
Description: Internet access for public subnet

Private Route Table (ejemplo con NAT):
text
Destination: 0.0.0.0/0
Target: NAT Gateway (production-nat)
Description: Acceso saliente desde subnet privada sin IP pública


3.5 Network Security: NSG y Security Lists
NSG: nsg-proxy (HTTP/HTTPS)
Propósito: tráfico web.
Reglas Ingress:
TCP 80 desde 0.0.0.0/0 (HTTP).
TCP 443 desde 0.0.0.0/0 (HTTPS).
NSG: nsg-ssh (SSH restringido)
Propósito: acceso SSH controlado.
Reglas Ingress:
TCP 22 desde 81.184.208.8/32 (IP autorizada).
NSG: nsg-internal (tráfico interno VCN)
Propósito: comunicación interna entre servicios.
Ingress:
Todo protocolo / puertos desde 10.1.0.0/16.
Egress:
Todo protocolo / puertos hacia 10.1.0.0/16.
Security Lists generales
Public Subnet:
Permitir 80/443 desde internet.
Private Subnet (DB):
Permitir puertos DB (ej. 5432, 3306, 1521) solo desde App Tier.

3.6 Gateways, VPN y NAT
Internet Gateway: igw-prod-frankfurt para subred pública.
NAT Gateway: para que privadas accedan a Internet sin IP pública.
VPN Site-to-Site (opcional):
DRG + CPE + IPSec Connection para conectar datacenter on-prem con OCI.
WireGuard:
Implementado en Bastion + VPN Server (Instance 3) para acceso seguro de admins a subred privada.

3.7 Direccionamiento IP
text
VCN Range: 10.1.0.0/16 (65,536 IPs)
├── Public Subnet: 10.1.1.0/24 (256 IPs)
│   ├── Instance 1: 10.1.1.x  (Apps + Proxy)
│   ├── Instance 3: 10.1.1.x  (Bastion + WireGuard)
│   └── Instance 4: 10.1.1.x  (Monitoring/Jumpbox)
└── Private Subnet: 10.1.2.0/24 (256 IPs)
    └── Instance 2: 10.1.2.x  (DB + Jobs)


💾 Almacenamiento
4.1 Object Storage
Buckets privados para:
Backups de BD.
Logs agregados.
Artifacts de CI/CD.
Ejemplo de bucket:
prod-data-bucket (compartment prod).
Tier inicial: Standard, con políticas de ciclo de vida:
text
Regla 1: 30 días → Infrequent Access (ahorro ~50%)
Regla 2: 90 días → Archive (ahorro ~80%)
Regla 3: 365 días → Eliminar (según compliance)

Encriptación:
Server-Side Encryption con Customer Managed Keys (KMS/Vault).
Replicación (opcional):
eu-frankfurt-1 → eu-madrid-1 (cross-region).

4.2 Block Storage (Volúmenes)
Para instancias A1 y E2:
Ejemplo:
prod-instance-data-volume – 500 GB, SSD, 60 VPUs, encrypted (KMS), backup diario.
Tipos de rendimiento (Basic / Balanced / High Performance) según IOPS y throughput.
Snapshots:
Snapshots diarios con retención 30 días para recuperación ante desastre.

4.3 File Storage (NFS)
Uso recomendado:
Directorios compartidos entre múltiples instancias (logs, artefactos shareados, etc.).
File System:
prod-shared-fs, 1 TB, balanced performance.
Mount Target en subnet privada; montaje NFS desde instancias (por ejemplo, para microservicios que comparten estado mínimo).

⚙️ Computación e Instancias
5.1 Resumen de Instancias (Proyecto GitOps)
Instance 1: Application + Proxy Server
Shape: VM.Standard.A1.Flex (ARM/Ampere).
2 OCPU, 12 GB RAM, ~80 GB Block Volume.
OS: Oracle Linux 9 (ARM64).
Red: Public Subnet (10.1.1.x) con IP pública.
NSGs: nsg-proxy, nsg-ssh, nsg-internal.
Propósito:
Frontend apps.
Reverse proxy (Nginx/Caddy).
Contenedores Docker (stack de microservicios).
Instance 2: Database + Jobs Server
Shape: VM.Standard.A1.Flex.
2 OCPU, 12 GB RAM, ~80 GB Block Volume.
OS: Oracle Linux 9 (ARM64).
Red: Private Subnet (10.1.2.x), sin IP pública.
NSGs: nsg-internal, nsg-ssh.
Propósito:
PostgreSQL / MySQL.
Cron jobs, backups, n8n workflows, servicios internos.
Instance 3: Bastion + VPN Server
Shape: VM.Standard.E2.1.Micro.
1 OCPU, 1 GB RAM, ~40 GB Block Volume.
OS: Oracle Linux 9.
Red: Public Subnet (10.1.1.x), IP pública.
NSGs: nsg-ssh, nsg-internal.
Propósito:
SSH bastion host.
WireGuard VPN.
Puerta de acceso segura a subred privada.
Instance 4: Monitoring + Secondary Jumpbox
Shape: VM.Standard.E2.1.Micro.
1 OCPU, 1 GB RAM, ~40 GB Block Volume.
OS: Oracle Linux 9.
Red: Public Subnet (10.1.1.x), IP pública opcional.
NSGs: nsg-ssh, nsg-internal.
Propósito:
Uptime monitoring (Uptime Kuma).
Jumpbox secundario.
Servicios ligeros de observabilidad.

5.2 Instancias Genéricas, Auto Scaling y OKE
Además de las instancias concretas de Always Free:
Compute Instances:
Imágenes tipo Ubuntu/Oracle Linux.
Scripts cloud-init para instalar Nginx, Docker, etc.
Auto Scaling Groups:
Ajustan el número de instancias según CPU, memoria, network o latencia.
Container Engine for Kubernetes (OKE):
Clusters con node pools sobre shapes A1 o AMD.
Integración con Load Balancer y Block Storage (CSI).
Ideal como evolución de la arquitectura Docker standalone hacia Kubernetes GitOps.

🗄️ Bases de Datos
6.1 Oracle Database Cloud Service (DBCS)
Oracle 19c Enterprise, HA con Data Guard (standby en otra AD o región).
Respaldos automáticos diarios, PITR hasta 35 días.
Almacenamiento en Object Storage encriptado.
6.2 MySQL Database Service
Versión 8.x, alta disponibilidad con réplicas.
Retención de backups (p.ej. 7 días).
Acceso desde subnet privada únicamente.
6.3 PostgreSQL
PostgreSQL 14 gestionado en compute o servicio administrado.
Ideal para microservicios (JSONB, CTE, particionado).
En la arquitectura GitOps, la Instance 2 alojará inicialmente PostgreSQL/MySQL en Docker sobre Oracle Linux 9, con backups a Object Storage y futura migración a servicio gestionado si se requiere.

🔒 Seguridad y Cumplimiento
7.1 Oracle Vault y KMS
Vault para secretos:
Contraseñas de BD.
API keys de terceros.
Certificados SSL.
KMS para claves de cifrado:
Clave maestra production-master-key (AES-256).
Rotación anual automática.
Acceso a secretos desde apps con SDK (Python, Node.js, etc.).

7.2 WAF y DDoS
WAF:
Reglas para SQL Injection, XSS, rate limiting y geo-blocking.
Reglas personalizadas para /admin/* y detección de scraping.
Protección DDoS:
Mitigación básica incluida en LB + WAF.
Opcional plan avanzado para cargas críticas.

7.3 Audit Logs y Cloud Guard
Audit Logs:
Inicios de sesión, cambios IAM, cambios de red, acceso a datos sensibles.
Cloud Guard:
Detecta anomalías (accesos inusuales, múltiples fallos de login, cambios críticos).
Puede lanzar respuestas automáticas (bloquear IP, generar incidentes, alerts).

📊 Monitoreo, Logging y Tracing
8.1 Monitoring
Métricas para:
Compute (CPU, memoria, disco, red).
Load balancers (latencia, errores, conexiones).
Bases de datos (TPS, lag, errores).
Dashboards:
Health check global.
Performance (P50/P95/P99).
Capacity planning (tendencias de CPU/memoria/almacenamiento).
Alertas:
CPU > 80%, error rate > 1%, DB down, storage > 85%, etc.
8.2 Logging
Centralización de logs:
Instancias (syslog, app logs).
LB (access/error).
WAF, BD, apps.
Envío a Object Storage (JSON comprimido), con retención hot/cold.
Integración con Grafana/Loki para visualización y queries.
8.3 Tracing Distribuido (APM)
Instrumentación con OpenTelemetry:
Java, Python, Node.js.
Visualización:
Spans por request (API Gateway → App → DB → Cache).
Detección de cuellos de botella (ej. queries lentas).

💰 Optimización de Costos y Always Free Tier
9.1 Always Free Tier (Proyecto GitOps)
ARM Compute: 4 OCPU / 24 GB RAM (Ampere A1).
AMD Micro: 2 × VM.Standard.E2.1.Micro.
Block Storage: 200 GB.
Load Balancer: 1 (10 Mbps).
1 IP pública (reservada).
9.2 Límites y Cuotas
Ejemplo de solicitud de aumento:
text
Resource: Cores for A1 DVH Instances
Region: eu-frankfurt-1
Current Limit: 0
Requested Limit: 4
Status: PENDING APPROVAL
Ticket: CAM-252231
Submitted: 2025-12-05 01:26:48 UTC
Expected Resolution: 1 business day

9.3 Estrategias de Ahorro (Global)
Reservas para instancias críticas (hasta ~30% de descuento).
Tiering automático en Object Storage.
Apagado programado de entornos de dev/lab.
Optimización de shapes (ARM vs AMD, burstable, etc.).
Consolidación de bases de datos cuando sea viable.

📈 Estado del Proyecto y Roadmap GitOps
10.1 Estado Actual
Completado:
 Suscripción a región Frankfurt.
 Creación de compartments (prod, lab, security).
 VCN y routing.
 Subnets pública y privada.
 Internet Gateway.
 Network Security Groups (3 NSGs).
 Route tables configuradas.
 Solicitud de aumento de límite A1 (CAM-252231).
En progreso:
 Aprobación de límite A1 cores.
 Despliegue de instancias compute.
Pendiente (GitOps & stack):
 Deployment de Instance 1 (Apps + Proxy).
 Deployment de Instance 2 (DB + Jobs).
 Deployment de Instance 3 (Bastion + WireGuard).
 Deployment de Instance 4 (Monitoring/Jumpbox).
 Configuración de scripts cloud-init.
 Instalación de Docker y docker-compose.
 Setup de PostgreSQL/MySQL, n8n, Nginx/Caddy.
 Configuración de WireGuard.
 Backups automatizados (Object Storage).
 Monitoreo (Uptime Kuma + OCI Monitoring).
 Flujo GitOps (repo Git + CI/CD).

🎯 Checklist de Implementación
text
□ Configuración Inicial
  □ Crear tenencia y habilitar regiones
  □ Configurar zona horaria y moneda

□ Identidad y Acceso
  □ Crear grupos (administrators, developers, operators, auditors)
  □ Crear usuarios y asignar grupos
  □ Habilitar MFA
  □ Crear API keys con rotación

□ Redes
  □ Crear VCN (vcn-prod-frankfurt / 10.1.0.0/16)
  □ Crear subred pública (10.1.1.0/24)
  □ Crear subred privada (10.1.2.0/24)
  □ Configurar Internet Gateway y, si aplica, NAT Gateway
  □ Definir NSGs (nsg-proxy, nsg-ssh, nsg-internal)
  □ Configurar route tables

□ Almacenamiento
  □ Crear buckets de Object Storage con políticas de ciclo de vida
  □ Crear volúmenes de Block Storage para instancias
  □ Configurar snapshots automáticos
  □ (Opcional) File Storage para NFS

□ Computación
  □ Desplegar instancias A1 y E2 según diseño
  □ Configurar Oracle Linux 9 / Ubuntu
  □ Añadir scripts cloud-init
  □ Instalar Docker y docker-compose

□ Bases de Datos
  □ Levantar PostgreSQL/MySQL (compute o gestionado)
  □ Configurar backups a Object Storage
  □ Establecer usuarios y políticas de acceso

□ Seguridad
  □ Configurar Vault y KMS
  □ Habilitar WAF y Cloud Guard
  □ Revisar NSG/Security Lists con principio de mínimo privilegio
  □ Configurar DDoS Protection (según criticidad)

□ Monitoreo y Logging
  □ Configurar dashboards (Monitoring)
  □ Definir alertas críticas
  □ Centralizar logs en Logging + Object Storage
  □ Implementar tracing distribuido en microservicios

□ Optimización de Costes
  □ Revisar uso de Always Free Tier
  □ Programar apagado de entornos no productivos
  □ Evaluar reservas y shapes ARM
  □ Establecer budgets y alertas de coste


📚 Recursos, Notas y Autor
Tecnologías del Proyecto GitOps
Cloud Provider: Oracle Cloud Infrastructure (OCI).
OS: Oracle Linux 9 (ARM64 + x86_64).
Compute: ARM Ampere A1 + AMD E2.1.Micro.
Containerización: Docker, Docker Compose.
Reverse Proxy: Nginx / Caddy.
Bases de datos: PostgreSQL / MySQL.
Automatización: n8n workflows.
VPN: WireGuard.
Monitoring: Uptime Kuma + OCI Monitoring/Logging.
IaC: Terraform (planificado).
GitOps: Repositorio Git + CI/CD (planificado).
¿Por qué Oracle Linux 9?
Optimizado para OCI, especialmente en ARM (Ampere).
Kernel UEK (Unbreakable Enterprise Kernel).
Compatibilidad con RHEL.
Soporte incluido sin coste adicional.
Parcheo y seguridad proactivos.
Recursos Oficiales
Documentación Oficial: 
https://docs.oracle.com/en-us/iaas/
Oracle Cloud Console: 
https://cloud.oracle.com
Oracle Cloud Status: 
https://status.oraclecloud.com
Comunidad: Oracle Cloud Community Forums
Autor
Mikel Apestegia
Email: mikelapestegia@gmail.com
Tenancy OCID: ocid1.tenancy.oc1..aaaaaaaam5ulr6q4pztpx3beybvoykiilojvmhjccn55rtbazrzoo4mdw7xq

Last Updated: 2025-12-05 02:00 CET
Infrastructure Status: 🟡 Awaiting A1 limit approval
Primary OCI Region: 🇩🇪 Frankfurt (eu-frankfurt-1)

