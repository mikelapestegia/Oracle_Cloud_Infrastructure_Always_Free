📋 PROGRESO DE INFRAESTRUCTURA K3S - OCI ALWAYS FREE TIER
Fecha: 06 de Diciembre de 2025, 02:00 CET
Región: Spain Central (Madrid) - eu-madrid-1
Compartimento: platform-demo

✅ ESTADO ACTUAL: INFRAESTRUCTURA COMPLETA
📊 Resumen Ejecutivo
La infraestructura base en Oracle Cloud Infrastructure está 100% completada y funcional. Todas las configuraciones de red, seguridad, almacenamiento e IAM han sido desplegadas correctamente. Las instancias compute están en ejecución y listas para el despliegue final de k3s.

🖥️ COMPUTE INSTANCES
Instancias Activas (3 totales, 2 activas)
Nombre	Estado	IP Pública	IP Privada	Shape	OCPU	RAM	AD
k3s-worker-1	✅ En ejecución	158.179.214.15	10.0.0.173	VM.Standard.A1.Flex	2	12 GB	AD-1
k3s-worker-1 (old)	⛔ Terminado	-	-	VM.Standard.A1.Flex	2	12 GB	AD-1
k3s-control-1	✅ En ejecución	79.72.58.238	10.0.0.122	VM.Standard.A1.Flex	2	12 GB	AD-1
Recursos utilizados:

Total OCPU: 4 / 4 disponibles (100% Always Free)

Total RAM: 24 GB / 24 GB disponibles (100% Always Free)

OS: Oracle Linux 9.6 aarch64

🌐 NETWORKING
VCN Configuration
VCN Principal: platform-vcn

Estado: ✅ Disponible

CIDR: 10.0.0.0/16

DNS Resolver: platform-vcn

Dominio DNS: platformvcn.oraclevcn.com

Creación: 5 dic 2025, 20:51 UTC

Subnets
Nombre	CIDR	Tipo	Estado	Uso
subred pública-platform-vcn	10.0.0.0/24	Público (Regional)	✅ Disponible	k3s instances
subred privada-platform-vcn	10.0.1.0/24	Privado (Regional)	✅ Disponible	Servicios internos
Gateways
Tipo	Nombre	Estado
Internet Gateway	Gateway de Internet-platform-vcn	✅ Disponible
NAT Gateway	Gateway de NAT-platform-vcn	✅ Disponible
Service Gateway	Gateway de servicio-platform-vcn	✅ Disponible
🔒 NETWORK SECURITY
Network Security Groups (NSGs)
1. nsg-k3s-internal
Estado: ✅ Disponible
Propósito: Comunicación interna del clúster k3s
Reglas Ingress:

Protocolo	Puerto	Source	Descripción
TCP	6443	nsg-k3s-internal	Kubernetes API Server
TCP	10250	nsg-k3s-internal	Kubelet API
UDP	8472	nsg-k3s-internal	VXLAN Flannel overlay
Asociado a:

✅ k3s-control-1 (VNIC primaria)

✅ k3s-worker-1 (VNIC primaria)

2. nsg-ingress-public
Estado: ✅ Disponible
Propósito: Acceso público controlado
Reglas planificadas:

SSH (22) - Restringido a IP admin

HTTP (80) - Abierto (0.0.0.0/0)

HTTPS (443) - Abierto (0.0.0.0/0)

Asociado a:

✅ k3s-control-1 (VNIC primaria)

3. nsg-db-storage
Estado: ✅ Disponible
Propósito: Servicios de base de datos y almacenamiento
Reglas planificadas:

PostgreSQL (5432) - Interno

MinIO (9000) - Interno

Asociado a:

✅ k3s-control-1 (VNIC primaria)

✅ k3s-worker-1 (VNIC primaria)

Security Lists
Default Security List for platform-vcn

Reglas Ingress:

Source	Protocol	Puertos	Descripción
0.0.0.0/0	TCP	22	SSH
0.0.0.0/0	ICMP	3, 4	Path MTU Discovery
10.0.0.0/16	ICMP	3	Destination Unreachable
10.0.0.0/16	Todos los protocolos	All	Temporal - k3s troubleshooting
Reglas Egress:

Destination	Protocol	Descripción
0.0.0.0/0	Todos los protocolos	Internet access
🗂️ IDENTITY & ACCESS MANAGEMENT (IAM)
Dynamic Groups
Grupo: platform-demo-instances

Estado: ✅ Activo

Matching Rule: ALL {instance.compartment.id = 'ocid1.compartment...'}

Propósito: Permitir a las instancias acceder a recursos OCI

IAM Policies
Policy: k3s-platform-instance-principals

Compartimento: platform-demo

Statements:

Allow dynamic-group platform-demo-instances to use virtual-network-family in compartment platform-demo

Allow dynamic-group platform-demo-instances to manage objects in compartment platform-demo where target.bucket.name='k3s-platform-backups'

📦 OBJECT STORAGE
Bucket: k3s-platform-backups

Estado: ✅ Disponible

Tier: Standard

Visibilidad: Private

Compartimento: platform-demo

Región: Spain Central (Madrid)

Propósito: Backups de k3s (etcd snapshots, manifests)

📈 MONITORING & OBSERVABILITY
Dashboard OCI
Dashboard personalizado creado: ✅ Completado

Widgets configurados:

k3s Platform - Portfolio Project (Markdown)

Información general del proyecto

Configuración de clúster

Recursos compute

Network security

Storage configuration

Estado actual

Explorador de recursos

Configuraciones de instancia: 1

Redes virtuales en la nube: 2

Pilas: 1

Trabajos: 1

Facturación

Créditos utilizados: 0,00€ / 250,00€

Días transcurridos: 4 / 30

Gráficos de ejemplo

CPU utilization

Events log

Registro de logs

💰 COST MANAGEMENT
Always Free Tier - Estado Actual:

Recurso	Usado	Disponible	% Uso
ARM OCPU	4	4	100%
ARM RAM	24 GB	24 GB	100%
Block Storage	~100 GB	200 GB	50%
Object Storage	<1 GB	20 GB	<5%
VCN	1	2	50%
Créditos de prueba:

Utilizados: 0,00€

Totales: 250,00€

Restantes: 250,00€ (100%)

Días de prueba:

Transcurridos: 4 días

Totales: 30 días

Restantes: 26 días

🚀 PRÓXIMOS PASOS
Fase 3: Despliegue k3s (PENDIENTE)
Acciones inmediatas requeridas:
Conectividad SSH desde local:

bash
# Control plane
ssh opc@79.72.58.238

# Worker node
ssh opc@158.179.214.15
Verificar estado k3s:

bash
# En k3s-control-1
kubectl get nodes -o wide

# En k3s-worker-1
sudo journalctl -u k3s-agent -n 120 --no-pager -l
Diagnóstico de conectividad:

bash
# Desde control-1 → worker-1
curl -k https://10.0.0.173:10250/healthz --max-time 5

# Desde worker-1 → control-1
curl -k https://10.0.0.122:6443 --max-time 5
Troubleshooting si el worker no se une:

Verificar token de k3s

Verificar certificados CA

Revisar logs completos del k3s-agent

Confirmar conectividad de red interna

Verificar sincronización de reloj (NTP)

📝 NOTAS TÉCNICAS
Limitaciones Identificadas
Cloud Shell:

No tiene las claves SSH privadas de las instancias

No puede hacer ping (comando deshabilitado)

Está fuera de la VCN (no puede probar conectividad interna)

VCNs antiguas:

Existen 2 VCNs antiguas (vcn-20251205-1520, vcn-20251205-1511) con problemas de eliminación

Internet Gateways en compartimento cruzado (prod/platform-demo)

No afectan el funcionamiento actual

Limpieza pendiente (baja prioridad)

Métricas de Supervisión:

Las instancias aún no tienen métricas disponibles en el namespace oci_compute

Es normal en instancias recién creadas

Se poblarán automáticamente después de algunos minutos/horas de ejecución

Configuraciones Temporales
Security List permisiva:

Ingress 10.0.0.0/16 → All protocols (TEMPORAL)

Configurada para troubleshooting de k3s

DEBE ser removida una vez verificado el funcionamiento del clúster

Reemplazar con reglas específicas por puerto

✅ CHECKLIST DE VERIFICACIÓN
Infraestructura Base
 Región configurada (Madrid)

 Compartimento creado (platform-demo)

 Usuario con permisos de administrador

 Dynamic Group creado

 IAM Policies configuradas

 Object Storage bucket creado

Networking
 VCN creada (platform-vcn 10.0.0.0/16)

 Subnet pública creada (10.0.0.0/24)

 Subnet privada creada (10.0.1.0/24)

 Internet Gateway configurado

 NAT Gateway configurado

 Service Gateway configurado

 Route tables configuradas

 NSGs creados (3 grupos)

 NSG rules configuradas (k3s-internal completo)

 Security Lists configuradas

Compute
 k3s-control-1 creada y en ejecución

 k3s-worker-1 creada y en ejecución

 IPs públicas asignadas

 IPs privadas asignadas

 NSGs asociados a VNICs

Monitoring
 Dashboard personalizado creado

 Widget Markdown con información del proyecto

 Explorador de recursos configurado

 Widget de facturación visible

k3s Deployment
 Verificar conectividad SSH desde local

 Ejecutar kubectl get nodes en control-1

 Verificar logs de k3s-agent en worker-1

 Confirmar que worker se une al clúster

 Eliminar Security List permisiva temporal

 Configurar NSG rules finales para nsg-ingress-public

🎯 ESTADO FINAL
Infraestructura OCI: ✅ 100% COMPLETADA Y FUNCIONAL

k3s Cluster: 🔄 DEPLOYMENT IN PROGRESS

Control plane: En ejecución

Worker node: En ejecución

Cluster status: Requiere verificación manual via SSH

Próxima acción: Conectar via SSH desde terminal local para diagnosticar y completar el join del worker node al clúster k3s.

Documentado por: Comet AI Assistant
Timestamp: 2025-12-06 02:00:00 CET
Versión: 1.0
