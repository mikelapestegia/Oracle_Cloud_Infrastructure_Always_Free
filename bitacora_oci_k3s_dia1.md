# 🛰️ Bitácora — Plataforma OCI + k3s (Oracle Linux)  
**Fecha:** 2025-12-06 (Europe/Madrid)  
**Objetivo del proyecto:** construir una plataforma cloud-native en OCI (k3s) con entrada HTTPS, observabilidad, GitOps y backups, para portfolio técnico orientado a roles 50–60k€.

---

## ✅ Estado actual (hoy)
### Clúster Kubernetes
- **Distribución:** k3s `v1.33.6+k3s1`
- **OS:** Oracle Linux 9.7 (ARM64)
- **Nodos:**
  - `k3s-control-1` (**Ready**) — Internal IP `10.0.0.122` — Public IP `79.72.58.238`
  - `k3s-worker-1` (**Ready**) — Internal IP `10.0.0.173`
- **Componentes base OK:**
  - `coredns`, `local-path-provisioner`, `metrics-server` en `kube-system`

Comando de verificación:
```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## ✅ Ingress funcionando desde Internet (hito importante)
- **Ingress Controller:** ingress-nginx (Helm) ✅
- **Service type:** `NodePort`
  - HTTP: `30080/TCP`
  - HTTPS: `30443/TCP`
- **Status:**
```bash
kubectl -n ingress-nginx get pods,svc
```

### Prueba real desde el portátil (OK)
Se validó acceso desde Internet a la app de demo (`whoami`) a través de ingress:
```bash
curl http://79.72.58.238:30080
```
Respuesta confirmada con `Hostname: whoami-...` ✅

---

## ✅ Helm instalado y operativo (en control-plane)
- **Helm:** `v3.19.2` instalado en `/usr/local/bin/helm`
- Se corrigió el error de Helm (`localhost:8080`) fijando el kubeconfig de k3s.

Kubeconfig a usar en k3s:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc
```

---

## 🧠 Decisiones técnicas tomadas hoy
1. **No exponer el API server (6443) a Internet.**  
   - Se comprobó conectividad interna desde el worker: `https://10.0.0.122:6443/ping` devolvió `200`.
2. **Ingress en NodePort para avanzar rápido** (30080/30443) y luego migrar a 80/443.
3. **Host de pruebas para TLS:** se propone usar `nip.io` (DNS automático a la IP pública) para emitir certificados sin comprar dominio.

---

## 🔐 Networking / OCI (lo que sabemos que funciona)
- El tráfico interno entre nodos funciona (unión del worker + ping al API server).
- Los NSG internos están bien (k3s operando en 2 nodos).
- Puertos NodePort accesibles desde Internet (al menos `30080` ya probado).

**Pendiente de revisar/ajustar mañana en OCI:**
- Abrir **80/443** hacia `k3s-control-1` (y cerrar 30080/30443 cuando ya no sean necesarios).
- Mantener **SSH (22) restringido a tu IP pública**.

---

# 📌 Plan para mañana (paso a paso)
## Meta del día
✅ Tener **HTTPS real** funcionando (cert-manager + Let’s Encrypt) y acceso por **80/443** (no por NodePort público).

---

## 1) (OCI) Preparar puertos públicos “bonitos”
**En el NSG público de `k3s-control-1`:**
- Ingress TCP **80** desde `0.0.0.0/0`
- Ingress TCP **443** desde `0.0.0.0/0`
- SSH 22 → solo tu IP

> Mantener temporalmente 30080/30443 si siguen en uso durante la transición.

---

## 2) (Control-plane) Redirigir 80→30080 y 443→30443 (transición)
Esto permite usar 80/443 mientras el servicio sigue siendo NodePort.

```bash
sudo bash -c 'cat >/etc/systemd/system/ingress-port-forward.service <<EOF
[Unit]
Description=Forward 80/443 to ingress-nginx NodePorts
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c "iptables -t nat -C PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 30080 2>/dev/null || iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 30080"
ExecStart=/bin/bash -c "iptables -t nat -C PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 30443 2>/dev/null || iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 30443"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now ingress-port-forward
```

Prueba:
```bash
curl -i http://79.72.58.238/
```

---

## 3) Instalar cert-manager
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create namespace cert-manager || true

helm upgrade --install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --set crds.enabled=true

kubectl -n cert-manager get pods
```

---

## 4) Crear ClusterIssuer (staging)
Sustituir `<TU_EMAIL>`:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: <TU_EMAIL>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

---

## 5) TLS con dominio automático (nip.io)
Usaremos:
- **Host:** `79.72.58.238.nip.io`

Actualizar Ingress de `whoami` para TLS:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - 79.72.58.238.nip.io
    secretName: whoami-tls
  rules:
  - host: 79.72.58.238.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              number: 80
EOF
```

Verificar emisión:
```bash
kubectl -n demo get certificate,order,challenge
kubectl -n demo describe certificate whoami-tls || true
```

Probar:
```bash
curl -i https://79.72.58.238.nip.io/
```

---

## 6) Pasar de staging a producción (Let’s Encrypt prod)
Crear issuer prod (cambia el `server`):
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <TU_EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Cambiar anotación del Ingress a `letsencrypt-prod`:
```bash
kubectl -n demo annotate ingress whoami cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite
```

---

## 7) Limpieza / endurecimiento (cierre del día)
- En OCI: **cerrar 30080/30443** a Internet (dejar solo 80/443).
- Mantener `6443` solo interno.
- SSH 22 restringido a tu IP.

---

# 🗺️ Próximos “bloques” tras HTTPS (para el roadmap)
1. **Observabilidad:** kube-prometheus-stack + Loki (dashboards + alertas).
2. **GitOps:** Argo CD (aplicaciones desde repo).
3. **Identidad:** Keycloak + Postgres (RBAC real).
4. **Backups:** bucket Object Storage + cronjobs (pg_dump + artefactos) + snapshots k3s.

---

## Anexo — comandos de salud rápida
```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n ingress-nginx get pods,svc
kubectl -n demo get pods,svc,ingress
sudo systemctl status k3s --no-pager
```
