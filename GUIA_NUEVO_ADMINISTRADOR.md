<div align="center">

# 🚀 Guía para el Nuevo Administrador
## Plataforma k3s en Oracle Cloud Infrastructure

![OCI](https://img.shields.io/badge/Oracle%20Cloud-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![k3s](https://img.shields.io/badge/k3s-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)

**Región:** Spain Central (Madrid) · **Compartimento:** `platform-demo`  
**Última actualización:** 09 Diciembre 2025

---

</div>

## 📋 Índice

- [🎯 ¿De qué trata este proyecto?](#-de-qué-trata-este-proyecto)
- [🏗️ Arquitectura actual](#️-arquitectura-actual)
- [✅ Infraestructura implementada](#-infraestructura-implementada)
- [⚙️ Servicios Kubernetes desplegados](#️-servicios-kubernetes-desplegados)
- [🔑 Acceso a la plataforma](#-acceso-a-la-plataforma)
- [📝 Pendiente de implementar](#-pendiente-de-implementar)
- [🚀 Cómo desplegar tu aplicación web](#-cómo-desplegar-tu-aplicación-web)
- [📚 Recursos útiles](#-recursos-útiles)

---

## 🎯 ¿De qué trata este proyecto?

Este proyecto es una **plataforma cloud-native completa** construida sobre **Oracle Cloud Infrastructure (OCI)** utilizando el tier **Always Free**. Proporciona infraestructura de producción para alojar aplicaciones web con características empresariales.

### 🌟 Características principales

| Estado | Componente | Descripción |
|--------|-----------|-------------|
| ✅ | **Clúster k3s** | Kubernetes ligero v1.33.6 multinodo |
| ✅ | **Ingress Controller** | ingress-nginx funcionando con NodePort |
| ✅ | **Networking OCI** | VCN, subnets, NSGs configurados |
| ✅ | **Aplicación demo** | whoami expuesta vía ingress |
| 🔄 | **TLS automático** | cert-manager + Let's Encrypt (en progreso) |
| 📋 | **GitOps** | Argo CD (planificado) |
| 📋 | **Observabilidad** | Prometheus + Grafana + Loki (planificado) |
| 📋 | **Autenticación** | Keycloak (planificado) |
| 📋 | **Backups** | OCI Object Storage (planificado) |

### 🎓 Objetivo del portfolio

Demostrar competencias profesionales en:
- ☁️ Administración de infraestructura cloud (OCI)
- ⚓ Orquestación de contenedores (Kubernetes/k3s)
- 🔒 Networking y seguridad (VCN, NSG, firewalls)
- 🔄 DevOps y automatización (IaC, CI/CD)
- 📊 Observabilidad y monitorización

---

## 🏗️ Arquitectura actual

### 🗺️ Diagrama de red

```
                      ☁️ Internet
                           |
                           ↓
                  ┌─────────────────┐
                  │ Internet Gateway│
                  └────────┬────────┘
                           |
         ┌──────────────────────────────────────────────────────────┐
         │ 🌐 VCN: platform-vcn (10.0.0.0/16)                │
         └────────────────────────────┬─────────────────────────────┘
                                      |
         ┌──────────────────────────────┴────────────────────────────────────────┐
         │  ☁️ Subnet pública: 10.0.0.0/24                      │
         │                                                        │
         │  ┌──────────────────────────────────────────────┐  │
         │  │  🖥️ k3s-control-1                           │  │
         │  │  • IP Pública: 79.72.58.238                   │  │
         │  │  • IP Privada: 10.0.0.122                     │  │
         │  │  • k3s server + API                          │  │
         │  │  • ingress-nginx (30080/30443)              │  │
         │  └──────────────────────────────────────────────┘  │
         │                                                        │
         │  ┌──────────────────────────────────────────────┐  │
         │  │  🖥️ k3s-worker-1                            │  │
         │  │  • IP Pública: 158.179.214.15                │  │
         │  │  • IP Privada: 10.0.0.173                     │  │
         │  │  • k3s agent (workloads)                     │  │
         │  └──────────────────────────────────────────────┘  │
         └───────────────────────────────────────────────────────┘
                                      |
         ┌──────────────────────────────┴────────────────────────────────────────┐
         │  🔒 Subnet privada: 10.0.1.0/24                      │
         │  (Servicios internos futuros)                          │
         └────────────────────────────────────────────────────────────┘
```

### 🧩 Componentes Kubernetes

- **Control Plane:** API server, scheduler, controller-manager
- **Worker Nodes:** Ejecutan pods de aplicaciones
- **Ingress:** Enrutamiento HTTP/HTTPS desde Internet
- **Storage:** local-path-provisioner (volúmenes persistentes)
- **DNS:** CoreDNS (resolución interna)
- **Métricas:** metrics-server (HPA, métricas de recursos)

---

## ✅ Infraestructura implementada

### 🖥️ Recursos Compute (Always Free)

| Instancia | 🟽 Estado | IP Pública | IP Privada | vCPU | RAM | Disco |
|-----------|---------|------------|------------|------|-----|-------|
| **k3s-control-1** | 🟢 Running | `79.72.58.238` | `10.0.0.122` | 2 | 12 GB | ~50 GB |
| **k3s-worker-1** | 🟢 Running | `158.179.214.15` | `10.0.0.173` | 2 | 12 GB | ~50 GB |

**Shape:** `VM.Standard.A1.Flex` (ARM64)  
**OS:** Oracle Linux 9.7  
**Uso Always Free:** 🔴 100% (4 vCPU / 24 GB RAM)

---

### 🌐 Networking (OCI)

#### VCN Principal: `platform-vcn`
```
CIDR: 10.0.0.0/16
DNS Domain: platformvcn.oraclevcn.com
```

#### Subnets

| Nombre | CIDR | Tipo | Uso |
|--------|------|------|-----|
| `subred pública-platform-vcn` | `10.0.0.0/24` | ☁️ Pública (Regional) | Nodos k3s con IPs públicas |
| `subred privada-platform-vcn` | `10.0.1.0/24` | 🔒 Privada (Regional) | Servicios internos |

#### Gateways

| Tipo | Estado | Función |
|------|--------|---------|
| 🌐 **Internet Gateway** | ✅ Disponible | Acceso bidireccional a Internet |
| 🔄 **NAT Gateway** | ✅ Disponible | Salida a Internet para subnet privada |
| 🔧 **Service Gateway** | ✅ Disponible | Acceso a servicios OCI (Object Storage) |

---

### 🔒 Seguridad (Network Security Groups)

#### 🔐 `nsg-k3s-internal` - Comunicación interna del clúster

| Protocolo | Puerto | Source | Descripción |
|-----------|--------|--------|-------------|
| TCP | 6443 | nsg-k3s-internal | 🎯 Kubernetes API Server |
| TCP | 10250 | nsg-k3s-internal | 🔧 Kubelet API |
| UDP | 8472 | nsg-k3s-internal | 🌐 VXLAN Flannel overlay |

**Asociado a:** k3s-control-1, k3s-worker-1

---

#### 🌍 `nsg-ingress-public` - Acceso público controlado

| Protocolo | Puerto | Source | Descripción |
|-----------|--------|--------|-------------|
| TCP | 22 | `[TU_IP]` | 🔑 SSH restringido (cambiar por tu IP) |
| TCP | 80 | `0.0.0.0/0` | 🌐 HTTP público |
| TCP | 443 | `0.0.0.0/0` | 🔒 HTTPS público |
| TCP | 30080 | `0.0.0.0/0` | 🚧 NodePort HTTP (temporal) |
| TCP | 30443 | `0.0.0.0/0` | 🚧 NodePort HTTPS (temporal) |

**Asociado a:** k3s-control-1

> ⚠️ **Nota:** Los puertos 30080/30443 son temporales mientras se completa la configuración TLS.

---

## 🚀 Cómo desplegar tu aplicación web

Esta es la sección más importante para ti como nuevo administrador. Aquí aprenderás a desplegar tu propia aplicación web en la plataforma.

### 📦 Paso 1: Contenarizar tu aplicación

Tu aplicación debe estar en un contenedor Docker. Si aún no lo está:

#### Ejemplo: Dockerfile para aplicación Node.js

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

#### Construir y publicar imagen

```bash
# Construir imagen
docker build -t tu-usuario/mi-app:v1.0 .

# Publicar a Docker Hub (o registro privado)
docker login
docker push tu-usuario/mi-app:v1.0
```

---

### 📝 Paso 2: Crear manifests de Kubernetes

Crea un directorio para tu aplicación:

```bash
mkdir -p k8s/apps/mi-app
cd k8s/apps/mi-app
```

#### 📄 Archivo: `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mi-app
  labels:
    name: mi-app
```

#### 📄 Archivo: `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  namespace: mi-app
  labels:
    app: mi-app
spec:
  replicas: 2  # Alta disponibilidad
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
      - name: mi-app
        image: tu-usuario/mi-app:v1.0
        ports:
        - containerPort: 3000
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        env:
        - name: NODE_ENV
          value: "production"
        livenessProbe:
          httpGet:
            path: /health  # Tu endpoint de salud
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready  # Tu endpoint de readiness
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### 📄 Archivo: `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-app
  namespace: mi-app
spec:
  selector:
    app: mi-app
  ports:
  - protocol: TCP
    port: 80          # Puerto del servicio
    targetPort: 3000  # Puerto del contenedor
  type: ClusterIP     # Servicio interno
```

#### 📄 Archivo: `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-app
  namespace: mi-app
  annotations:
    # Para HTTPS automático (cuando cert-manager esté instalado)
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mi-app.79.72.58.238.nip.io  # Dominio temporal con nip.io
    secretName: mi-app-tls
  rules:
  - host: mi-app.79.72.58.238.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mi-app
            port:
              number: 80
```

---

### 🚀 Paso 3: Desplegar en el clúster

#### Conectar por SSH al control plane

```bash
ssh opc@79.72.58.238
```

#### Configurar kubectl

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

#### Subir los manifests al servidor

Desde tu máquina local:

```bash
# Copiar archivos al servidor
scp -r k8s/apps/mi-app opc@79.72.58.238:~/
```

#### Aplicar los manifests

En el control plane:

```bash
cd ~/mi-app

# Crear namespace
kubectl apply -f namespace.yaml

# Desplegar aplicación
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

---

### ✅ Paso 4: Verificar el despliegue

```bash
# Ver pods
kubectl -n mi-app get pods

# Ver servicio
kubectl -n mi-app get svc

# Ver ingress
kubectl -n mi-app get ingress

# Ver logs
kubectl -n mi-app logs -l app=mi-app --tail=50 -f

# Describir pod (troubleshooting)
kubectl -n mi-app describe pod <nombre-pod>
```

#### Verificación esperada

```bash
$ kubectl -n mi-app get pods
NAME                      READY   STATUS    RESTARTS   AGE
mi-app-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
mi-app-xxxxxxxxxx-yyyyy   1/1     Running   0          2m
```

---

### 🌐 Paso 5: Probar tu aplicación

#### Acceso HTTP (temporal con NodePort)

```bash
# Desde tu máquina local
curl http://79.72.58.238:30080 -H "Host: mi-app.79.72.58.238.nip.io"
```

#### Acceso desde navegador

```
http://79.72.58.238:30080
```

Asegúrate de enviar el header `Host: mi-app.79.72.58.238.nip.io` o espera a que TLS esté configurado para usar el dominio completo.

---

### 🔒 Paso 6: Habilitar HTTPS (cuando cert-manager esté listo)

Una vez que cert-manager esté instalado:

```bash
# El certificado se creará automáticamente
kubectl -n mi-app get certificate
kubectl -n mi-app describe certificate mi-app-tls

# Esperar a que el estado sea "Ready"
kubectl -n mi-app get certificate -w
```

Acceder vía HTTPS:

```
https://mi-app.79.72.58.238.nip.io
```

---

## 📚 Recursos útiles

### Documentación oficial

- [k3s Documentation](https://docs.k3s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager](https://cert-manager.io/docs/)
- [OCI Documentation](https://docs.oracle.com/en-us/iaas/)

### Comandos útiles

#### Ver estado del clúster

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl top nodes
kubectl top pods -A
```

#### Troubleshooting

```bash
# Logs de un pod
kubectl -n <namespace> logs <pod-name> -f

# Logs de todos los pods de un deployment
kubectl -n <namespace> logs -l app=<app-name> --tail=100 -f

# Ejecutar comando en un pod
kubectl -n <namespace> exec -it <pod-name> -- /bin/sh

# Describir recurso
kubectl -n <namespace> describe pod/<pod-name>
kubectl -n <namespace> describe ingress/<ingress-name>

# Ver eventos del clúster
kubectl get events --sort-by='.lastTimestamp' -A
```

#### Gestión de recursos

```bash
# Escalar deployment
kubectl -n <namespace> scale deployment/<deployment-name> --replicas=3

# Reiniciar deployment
kubectl -n <namespace> rollout restart deployment/<deployment-name>

# Ver historial de rollouts
kubectl -n <namespace> rollout history deployment/<deployment-name>

# Rollback
kubectl -n <namespace> rollout undo deployment/<deployment-name>
```

### Archivos importantes en el servidor

```bash
# Kubeconfig de k3s
/etc/rancher/k3s/k3s.yaml

# Configuración de k3s
/etc/systemd/system/k3s.service

# Logs de k3s
journalctl -u k3s -f
journalctl -u k3s-agent -f  # En workers

# Manifests de k3s
/var/lib/rancher/k3s/server/manifests/
```

---

## 👏 ¡Éxito!

Ahora tienes toda la información necesaria para administrar la plataforma y desplegar tus aplicaciones web.

Para cualquier duda, revisa:
- 📝 Las bitácoras en `docs/bitacora/`
- 👁️ El README principal del proyecto
- 💻 La documentación oficial de k3s y Kubernetes

**¡Buena suerte con tus despliegues! 🚀**

---

<div align="center">

🛠️ **Creado por:** Mikel Apestegia  
💻 **Portfolio:** [Oracle Cloud Infrastructure Always Free Platform](https://github.com/mikelapestegia/Oracle_Cloud_Infrastructure_Always_Free)

</div>
