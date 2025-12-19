<div align="center">

# 🚀 Huellas del Norte - Cloud Platform
### Plataforma Cloud-Native de Producción en OCI Always Free

[![OCI](https://img.shields.io/badge/Oracle%20Cloud-F80000?style=for-the-badge&logo=oracle&logoColor=white)](https://cloud.oracle.com/)
[![Kubernetes](https://img.shields.io/badge/kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![k3s](https://img.shields.io/badge/k3s-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)](https://k3s.io/)
[![React](https://img.shields.io/badge/react-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://react.dev/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://cloudflare.com/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![Cluster Status](https://img.shields.io/badge/cluster-operational-success?style=for-the-badge)](https://huellasdelnorte.com)
[![Always Free](https://img.shields.io/badge/cost-$0%2Fmonth-brightgreen?style=for-the-badge)](https://www.oracle.com/cloud/free/)

**Infraestructura cloud empresarial + Web moderna con IA local**  
**Todo desplegado en Oracle Cloud Infrastructure - Tier Always Free**

[🌐 huellasdelnorte.com](https://huellasdelnorte.com) • [📚 Documentación](https://github.com/mikelapestegia/Oracle_Cloud_Infrastructure_Always_Free/blob/main/README.md#-tabla-de-contenidos) • [🚀 Quick Start](https://github.com/mikelapestegia/Oracle_Cloud_Infrastructure_Always_Free/blob/main/README.md#-quick-start) • [📊 Estado](https://github.com/mikelapestegia/Oracle_Cloud_Infrastructure_Always_Free/blob/main/README.md#-estado-del-proyecto)

</div>

---

## 🎯 Acerca del Proyecto

**Huellas del Norte** es una plataforma cloud-native completa que demuestra cómo construir una infraestructura de producción empresarial sin costo alguno, combinando:

- 📡 **Infraestructura cloud robusta** - Kubernetes (k3s) en OCI Always Free
- 🌐 **Aplicación web moderna** - React con chatbot de IA local desplegada en [huellasdelnorte.com](https://huellasdelnorte.com)
- 🔒 **Seguridad y GitOps** - TLS automático, Argo CD, monitoring completo
- 🤖 **Inteligencia Artificial** - Chatbot con modelo local ejecutándose en el cluster

### ✨ Características Destacadas

```mermaid
graph LR
    A[👥 Usuario] -->|HTTPS| B[Cloudflare]
    B -->|Proxy| C[🌐 huellasdelnorte.com]
    C -->|NodePort| D[Ingress Nginx]
    D --> E[React App]
    D --> F[🤖 AI Chatbot API]
    F --> G[Modelo IA Local]
    E -.-> H[Argo CD]
    H -.-> I[GitOps Repo]
```

---

## 📋 Tabla de Contenidos

- [🎯 Acerca del Proyecto](#-acerca-del-proyecto)
- [💡 Componentes Principales](#-componentes-principales)
  - [🌐 Aplicación Web - huellasdelnorte.com](#-aplicación-web---huellasdelnortecom)
  - [📡 Infraestructura Cloud](#-infraestructura-cloud)
  - [🤖 Chatbot con IA Local](#-chatbot-con-ia-local)
- [🏗️ Arquitectura](#%EF%B8%8F-arquitectura)
- [✅ Estado de Implementación](#-estado-de-implementación)
- [🚀 Quick Start](#-quick-start)
- [📊 Métricas y Monitoreo](#-métricas-y-monitoreo)
- [📚 Documentación](#-documentación)
- [🔧 Stack Tecnológico](#-stack-tecnológico)
- [💰 Costos y ROI](#-costos-y-roi)
- [📨 Roadmap](#-roadmap)
- [🤝 Contribuir](#-contribuir)

---

## 💡 Componentes Principales

### 🌐 Aplicación Web - huellasdelnorte.com

<div align="center">

[![Web Status](https://img.shields.io/website?url=https%3A%2F%2Fhuellasdelnorte.com&style=for-the-badge&label=huellasdelnorte.com)](https://huellasdelnorte.com)

</div>

**Sitio web moderno construido con React y desplegado en el cluster k3s**

#### ✨ Características de la Web

- ✅ **Framework**: React 18+ con Hooks modernos
- ✅ **Dominio**: `huellasdelnorte.com` gestionado por Cloudflare
- ✅ **SSL/TLS**: Certificados automáticos con cert-manager + Let's Encrypt  
- ✅ **CDN**: Cloudflare para optimización global y protección DDoS
- ✅ **Responsive**: Diseño adaptable a todos los dispositivos
- 🔄 **CI/CD**: Despliegue automático vía Argo CD

#### 🤖 Chatbot Inteligente

```yaml
Chatbot:
  Modelo: Ejecutado localmente en el cluster (ej: LLaMA, Mistral)
  Backend: API Python/Node.js en Kubernetes
  Frontend: Componente React integrado
  Características:
    - Respuestas en tiempo real
    - Contexto conversacional
    - Sin dependencias cloud externas
    - Privacidad total (datos no salen del cluster)
```

**Tecnologías del Chatbot:**
- 🧠 Modelo de IA local (ollama, llama.cpp, etc.)
- 🐍 API Backend en Python/FastAPI o Node.js
- ⚛️ Interfaz React con WebSocket para streaming
- 📊 Monitoring de uso y rendimiento
