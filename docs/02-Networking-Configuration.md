# 📡 Configuración de Red OCI - k3s Always Free

## 🎯 Objetivo

Este documento describe la configuración de red básica pero bien organizada para el cluster k3s en Oracle Cloud Infrastructure Always Free, diseñada para ser funcional desde el inicio y permitir endurecimiento progresivo sin necesidad de rediseñar.

## 🏗️ Arquitectura de Red

### Principios de Diseño

- **Simplicidad operativa**: Acceso SSH y web desde cualquier red (fase inicial)
- **Preparado para producción**: Estructura lista para endurecer sin romper
- **Separación lógica**: Subred pública/privada desde el inicio
- **NSG desde día 1**: Reglas por aplicación, no solo por subred

### Diagrama de Red

```
┌─────────────────────────────────────────────────────────────┐
│                Oracle Cloud Infrastructure                  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              VCN: k3s-vcn (10.0.0.0/16)                │ │
│  │                                                        │ │
│  │  ┌──────────────────────┐  ┌─────────────────────┐   │ │
│  │  │  Subred Pública      │  │  Subred Privada     │   │ │
│  │  │  10.0.0.0/24         │  │  10.0.1.0/24        │   │ │
│  │  │  (k3s-public-subnet) │  │  (k3s-private-subnet)│   │ │
│  │  │                      │  │                     │   │ │
│  │  │  ┌────────────────┐  │  │  ┌───────────────┐  │   │ │
│  │  │  │ Control Plane  │  │  │  │  Servicios    │  │   │ │
│  │  │  │ (IP pública)   │  │  │  │  Internos     │  │   │ │
│  │  │  │ E2.1.Micro     │  │  │  │  (futuro)     │  │   │ │
│  │  │  │                │  │  │  │               │  │   │ │
│  │  │  │ Worker Node    │  │  │  │  - Keycloak   │  │   │ │
│  │  │  │ (IP pública)   │  │  │  │  - PostgreSQL │  │   │ │
│  │  │  │ A1.Flex 24GB   │  │  │  │  - Monitoring │  │   │ │
│  │  │  └────────────────┘  │  │  └───────────────┘  │   │ │
│  │  │         │            │  │          │          │   │ │
│  │  └─────────┼────────────┘  └──────────┼──────────┘   │ │
│  │            │                           │              │ │
│  │      ┌─────▼──────┐            ┌──────▼────────┐     │ │
│  │      │ Internet   │            │ NAT Gateway   │     │ │
│  │      │ Gateway    │            │ (solo salida) │     │ │
│  │      └────────────┘            └───────────────┘     │ │
│  │                                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 📋 Configuración de Componentes

### VCN y Subredes

| Recurso            | Nombre                | Tipo     | CIDR          | Gateways / Route Table     | Comentario                                                                 |
|--------------------|-----------------------|----------|---------------|----------------------------|-----------------------------------------------------------------------------|
| VCN                | `k3s-vcn`            | VCN      | `10.0.0.0/16` | -                          | Red principal del cluster k3s                                              |
| Subred pública     | `k3s-public-subnet`  | Regional | `10.0.0.0/24` | RT: `k3s-public-rt` + IGW  | VMs con IP pública (control-plane, worker/ingress)                         |
| Subred privada     | `k3s-private-subnet` | Regional | `10.0.1.0/24` | RT: `k3s-private-rt` + NAT | VMs y servicios internos sin IP pública (futuro: Keycloak, DB, monitoring) |
| Route table pública| `k3s-public-rt`      | RT       | -             | → Internet Gateway         | Todo el tráfico saliente de la subred pública a Internet                   |
| Route table privada| `k3s-private-rt`     | RT       | -             | → NAT Gateway              | Salida a Internet desde la subred privada (sin entrada directa)           |
| Internet Gateway   | `k3s-igw`            | IGW      | -             | VCN `k3s-vcn`              | Acceso Internet para subred pública                                       |
| NAT Gateway        | `k3s-nat`            | NAT      | -             | VCN `k3s-vcn`              | Acceso saliente desde subred privada                                      |

### Security Lists

#### Subred Pública - Security List

| Dirección | Proto | Puerto(s)    | Origen/Destino | Descripción                                     |
|----------|-------|--------------|----------------|-------------------------------------------------|
| Ingress  | TCP   | 22           | `0.0.0.0/0`    | SSH a VMs públicas desde cualquier red          |
| Ingress  | TCP   | 30080-30443  | `0.0.0.0/0`    | NodePort HTTP/HTTPS para ingress-nginx          |
| Ingress  | TCP   | 6443         | `0.0.0.0/0`    | API k3s pública (modo demo)                     |
| Egress   | ALL   | ALL          | `0.0.0.0/0`    | Tráfico saliente libre                          |

#### Subred Privada - Security List

| Dirección | Proto | Puerto(s) | Origen/Destino | Descripción                                           |
|----------|-------|-----------|----------------|-------------------------------------------------------|
| Ingress  | ALL   | ALL       | `10.0.0.0/16`  | Solo tráfico interno desde el VCN                     |
| Egress   | ALL   | ALL       | `0.0.0.0/0`    | Salida a Internet vía NAT Gateway                     |

### Network Security Groups (NSG)

#### NSG: k3s-control-plane

| Regla       | Dirección | Proto | Puerto(s)    | Origen/Destino    | Comentario                                                        |
|------------|----------|-------|--------------|-------------------|-------------------------------------------------------------------|
| SSH        | Ingress  | TCP   | 22           | `0.0.0.0/0`       | Igual que ahora, acceso SSH desde cualquier red                   |
| API k3s    | Ingress  | TCP   | 6443         | `0.0.0.0/0`       | API k3s accesible desde cualquier sitio                          |
| Egress all | Egress   | ALL   | ALL          | `0.0.0.0/0`       | Salida libre                                                      |

#### NSG: k3s-worker-ingress

| Regla       | Dirección | Proto | Puerto(s)    | Origen/Destino    | Comentario                                                        |
|------------|----------|-------|--------------|-------------------|-------------------------------------------------------------------|
| NodePort   | Ingress  | TCP   | 30080-30443  | `0.0.0.0/0`       | HTTP/HTTPS de las apps                                            |
| Egress all | Egress   | ALL   | ALL          | `0.0.0.0/0`       | Salida libre                                                      |

## 📑 Configuración YAML (Referencia)

```yaml
oci_networking:
  vcn:
    name: k3s-vcn
    cidr_block: 10.0.0.0/16
    region: eu-madrid-1

  gateways:
    internet_gateway:
      name: k3s-igw
      enabled: true
    nat_gateway:
      name: k3s-nat
      enabled: true

  route_tables:
    public:
      name: k3s-public-rt
      rules:
        - destination: 0.0.0.0/0
          destination_type: CIDR_BLOCK
          target: internet_gateway:k3s-igw
    private:
      name: k3s-private-rt
      rules:
        - destination: 0.0.0.0/0
          destination_type: CIDR_BLOCK
          target: nat_gateway:k3s-nat

  subnets:
    public:
      name: k3s-public-subnet
      cidr_block: 10.0.0.0/24
      type: regional
      route_table: k3s-public-rt
      prohibit_public_ip_on_vnic: false
      usage:
        - k3s-control-plane
        - k3s-worker-ingress
    private:
      name: k3s-private-subnet
      cidr_block: 10.0.1.0/24
      type: regional
      route_table: k3s-private-rt
      prohibit_public_ip_on_vnic: true
      usage:
        - internal-services
        - future-databases

  security_lists:
    public_subnet_sl:
      applies_to: k3s-public-subnet
      ingress:
        - protocol: tcp
          port: 22
          source: 0.0.0.0/0
          description: SSH desde cualquier red
        - protocol: tcp
          port_range: "30080-30443"
          source: 0.0.0.0/0
          description: NodePort HTTP/HTTPS (ingress-nginx)
        - protocol: tcp
          port: 6443
          source: 0.0.0.0/0
          description: API k3s pública (modo demo)
      egress:
        - protocol: all
          destination: 0.0.0.0/0
          description: Salida a Internet

    private_subnet_sl:
      applies_to: k3s-private-subnet
      ingress:
        - protocol: all
          source: 10.0.0.0/16
          description: Tráfico solo desde dentro del VCN
      egress:
        - protocol: all
          destination: 0.0.0.0/0
          description: Salida a Internet vía NAT

  network_security_groups:
    k3s-control-plane:
      description: Reglas del nodo control-plane k3s
      rules:
        ingress:
          - protocol: tcp
            port: 22
            source: 0.0.0.0/0
            description: SSH a control-plane
          - protocol: tcp
            port: 6443
            source: 0.0.0.0/0
            description: API k3s accesible desde cualquier red (demo)
        egress:
          - protocol: all
            destination: 0.0.0.0/0
            description: Salida libre
            
    k3s-worker-ingress:
      description: Reglas del nodo worker que expone NodePort
      rules:
        ingress:
          - protocol: tcp
            port_range: "30080-30443"
            source: 0.0.0.0/0
            description: HTTP/HTTPS público (NodePort)
        egress:
          - protocol: all
            destination: 0.0.0.0/0
            description: Salida libre
```

## ✅ Mejores Prácticas Implementadas

### Fase Actual (Desarrollo/Demo)

- ✅ **VCN bien estructurado**: Separación subred pública/privada desde el inicio
- ✅ **Gateways correctos**: IGW para subred pública, NAT para privada
- ✅ **Route tables dedicadas**: Una por tipo de subred
- ✅ **NSG implementados**: Reglas por componente, no solo por subred
- ✅ **Nomenclatura consistente**: Todos los recursos siguen convención `k3s-*`
- ✅ **Documentación**: Configuración como código (YAML) y tablas de referencia

### Seguridad Actual (Modo Demo)

⚠️ **NOTA**: Esta configuración permite acceso desde cualquier red (`0.0.0.0/0`) para facilitar el desarrollo inicial. Ver sección "Plan de Endurecimiento" para producción.

**Puertos expuestos actualmente:**
- TCP 22 (SSH) - Acceso administrativo
- TCP 6443 (k3s API) - Gestión del cluster
- TCP 30080-30443 (NodePort) - Tráfico HTTP/HTTPS de aplicaciones

## 🔐 Plan de Endurecimiento Futuro

Cuando el cluster esté estable y en producción, seguir estos pasos:

### Paso 1: Restringir SSH

**Cambiar:**
```yaml
# De:
source: 0.0.0.0/0

# A:
source: TU_IP_PUBLICA/32  # o rango de tu organización
```

**Cómo:**
1. Ir a NSG `k3s-control-plane`
2. Editar regla "SSH"
3. Cambiar origen de `0.0.0.0/0` a tu IP pública
4. Repetir para security list de subred pública

### Paso 2: Proteger API k3s

**Opción A - Restringir por IP:**
```yaml
# API k3s solo desde IPs conocidas
source: TU_IP_PUBLICA/32
```

**Opción B - Mover a privada (recomendado para producción):**
1. Crear bastión en subred pública
2. Mover control-plane a subred privada
3. Acceder a API solo vía túnel SSH desde bastión

### Paso 3: Implementar Bastión (Opcional)

**Ventajas:**
- Un solo punto de entrada SSH
- Control-plane y worker pueden moverse a subred privada
- Auditoría centralizada de accesos

**Configuración:**
1. Crear VM E2.1.Micro adicional en subred pública
2. NSG `k3s-bastion`:
   - Ingress: TCP 22 desde tu IP
   - Egress: TCP 22 hacia subred privada
3. Acceso a control-plane:
   ```bash
   ssh -J opc@bastion opc@control-plane-private-ip
   ```

### Paso 4: Endurecer NodePort

**Opción A - CloudFlare (gratis):**
1. Proxy tu dominio por CloudFlare
2. Restringir NSG a solo IPs de CloudFlare
3. Beneficios: DDoS protection, CDN, WAF básico

**Opción B - Geoblocking:**
```yaml
# Solo tráfico desde España/UE (requiere WAF de pago)
source: RANGOS_IP_PAIS
```

### Paso 5: Mover Servicios a Subred Privada

**Servicios que deben ir a privada:**
- PostgreSQL (Keycloak)
- Prometheus/Grafana (acceso vía túnel o OAuth proxy)
- Loki
- ArgoCD (acceso vía ingress con autenticación)

**Acceso desde pública:**
- Solo vía ingress-nginx con TLS
- Autenticación con Keycloak/OAuth2
- NetworkPolicy en Kubernetes para restringir pods

## 🛠️ Implementación en OCI Console

### Crear VCN y Subredes

1. **Networking** → **Virtual Cloud Networks** → **Start VCN Wizard**
2. Seleccionar **"VCN with Internet Connectivity"**
3. Configurar:
   - VCN Name: `k3s-vcn`
   - VCN CIDR: `10.0.0.0/16`
   - Public Subnet CIDR: `10.0.0.0/24`
   - Private Subnet CIDR: `10.0.1.0/24`
4. Marcar **"Use DNS hostnames in this VCN"**
5. Click **"Next"** → **"Create"**

### Configurar Security Lists

#### Subred Pública:

1. Ir a VCN → **k3s-public-subnet** → **Default Security List**
2. **Add Ingress Rules**:
   - Rule 1: Source `0.0.0.0/0`, Protocol `TCP`, Port `22`
   - Rule 2: Source `0.0.0.0/0`, Protocol `TCP`, Port range `30080-30443`
   - Rule 3: Source `0.0.0.0/0`, Protocol `TCP`, Port `6443`

#### Subred Privada:

1. Ir a VCN → **k3s-private-subnet** → **Default Security List**
2. **Add Ingress Rules**:
   - Rule 1: Source `10.0.0.0/16`, Protocol `All`

### Crear Network Security Groups

1. **Networking** → **Virtual Cloud Networks** → **k3s-vcn** → **Network Security Groups**
2. Click **"Create Network Security Group"**

#### NSG: k3s-control-plane

**Security Rules:**
- Ingress, TCP, Port 22, Source `0.0.0.0/0`
- Ingress, TCP, Port 6443, Source `0.0.0.0/0`
- Egress, All Protocols, Destination `0.0.0.0/0`

#### NSG: k3s-worker-ingress

**Security Rules:**
- Ingress, TCP, Port range `30080-30443`, Source `0.0.0.0/0`
- Egress, All Protocols, Destination `0.0.0.0/0`

### Asociar NSG a Instancias

1. **Compute** → **Instances** → Seleccionar instancia
2. **Attached VNICs** → Click en VNIC
3. **Network Security Groups** → **Edit**
4. Añadir NSG correspondiente

## 📊 Estado de Implementación

| Componente           | Estado       | Fecha       | Notas                                  |
|---------------------|--------------|-------------|----------------------------------------|
| VCN                 | ✅ Operacional | 2025-12-19 | CIDR 10.0.0.0/16                      |
| Subred Pública     | ✅ Operacional | 2025-12-19 | 10.0.0.0/24 con IGW                   |
| Subred Privada      | ✅ Operacional | 2025-12-19 | 10.0.1.0/24 con NAT                   |
| Security Lists      | ✅ Configurado | 2025-12-19 | Reglas básicas implementadas          |
| NSG Control-Plane   | 🟡 Pendiente   | -           | A crear según documentación          |
| NSG Worker-Ingress  | 🟡 Pendiente   | -           | A crear según documentación          |
| Endurecimiento      | 🟡 Futuro     | -           | Cuando cluster esté en producción    |

## 📚 Referencias

- [OCI Networking Best Practices](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/bestpractices.htm)
- [Security Lists vs NSGs](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm)
- [k3s Network Requirements](https://docs.k3s.io/installation/requirements#networking)

---

📄 **Documento**: 02-Networking-Configuration.md  
📅 **Última actualización**: 2025-12-19  
✍️ **Autor**: Mikel Apestegia  
📝 **Proyecto**: [Oracle_Cloud_Infrastructure_Always_Free](https://github.com/mikelapestegia/Oracle_Cloud_Infrastructure_Always_Free)
