# 🐶 Tienda de Alimentos para Perritos — EFT ISY1101

Proyecto de la Evaluación Final Transversal de **Introducción a Herramientas DevOps (ISY1101)** — Duoc UC, 2025.

**Integrantes:** Mario Graterón y Álvaro Riveros

Aplicación CRUD de productos ("Tienda de Alimentos para Perritos"), con el ciclo completo de integración y entrega continua (CI/CD) automatizado con GitHub Actions, contenedores Docker, y despliegue en un clúster **Amazon EKS** orquestado con Kubernetes.

---

## 📐 Arquitectura

```
                        ┌─────────────────────────────┐
                        │        Amazon EKS            │
                        │      (EPFtienda-cluster)     │
                        │                               │
   Internet ──► CLB ──► │  ┌─────────┐   ┌───────────┐ │
                        │  │ frontend│──►│  backend  │ │
                        │  │ (nginx) │   │ (Node.js) │ │
                        │  └─────────┘   └─────┬─────┘ │
                        │                       │       │
                        │                 ┌─────▼─────┐ │
                        │                 │  MySQL 8  │ │
                        │                 │ (tienda-db)│ │
                        │                 └───────────┘ │
                        └─────────────────────────────┘
```

- **Frontend:** HTML/JS estático, servido con **Nginx**, que hace proxy de `/api/` hacia el backend (`default.conf`).
- **Backend:** **Node.js + Express**, expone la API REST `/api/productos` (CRUD completo) y `/api/health` para los probes de Kubernetes. Conecta a MySQL usando `mysql2`.
- **Base de datos:** **MySQL 8**, con `init.sql` que crea el esquema `productos` y datos de ejemplo automáticamente al iniciar el contenedor.
- **Orquestación:** Kubernetes (namespace `tienda`), con **Horizontal Pod Autoscaler (HPA)** para backend y frontend.
- **Exposición pública:** Classic Load Balancer (ver sección de infraestructura).
- **CI/CD:** GitHub Actions, build → push a Amazon ECR → deploy automatizado a EKS.

---

## 🗂️ Estructura del repositorio

```
tienda-perritos-devops/
├── .github/workflows/
│   └── deploy-eks.yml       # Pipeline CI/CD
├── frontend/
│   ├── Dockerfile
│   ├── default.conf         # Config nginx (proxy /api/)
│   ├── index.html
│   └── app.js
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── db/
│   ├── Dockerfile
│   └── init.sql
├── k8s/
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
├── docker-compose.yml       # Orquestación local (desarrollo)
└── README.md
```

---

## 🚀 Cómo levantar el proyecto en local (Docker Compose)

Requisitos: Docker Desktop instalado.

```bash
git clone https://github.com/MarioGrateronDuoc/tienda-perritos-devops.git
cd tienda-perritos-devops
docker compose up --build
```

Esto levanta 3 contenedores en red interna (`tienda-net`):
- `tienda-db` (MySQL 8, puerto `3306`)
- `tienda-backend` (Node.js, puerto `3001`)
- `tienda-frontend` (Nginx, puerto `8080` → mapea al `80` interno)

Una vez levantado, abre en el navegador:

```
http://localhost:8080
```

Para detener y limpiar:
```bash
docker compose down
```

---

## ☁️ Infraestructura en AWS

### Red (VPC) — creada manualmente
- **VPC:** `EPF-VPC` — `10.0.0.0/16`
- **4 subredes** en 2 zonas de disponibilidad (`us-east-1a`, `us-east-1b`):
  - 2 públicas: `tienda-public-1a`, `tienda-public-1b`
  - 2 privadas: `tienda-private-1a`, `tienda-private-1b`
- **Internet Gateway:** `EPFtienda-igw`
- **NAT Gateway:** `EPFtienda-nat` (con Elastic IP, en subred pública)
- **Route Tables:** `EPFTienda-public-rtc` (→ IGW), `EPFtienda-private-rt` (→ NAT)
- **Security Groups:**
  - `tienda-sg-cluster`: acceso HTTPS (443) al control plane de EKS
  - `tienda-sg-nodes`: tráfico interno entre nodos + NodePort (31585) + puerto 80 para el Load Balancer

### Clúster EKS
- **Nombre:** `EPFtienda-cluster`
- **Rol IAM:** `LabRole` (rol provisto por AWS Academy Learner Lab; el entorno no permite crear roles/políticas IAM personalizados)
- **Node Group administrado:** 2 instancias `t3.medium` en subredes públicas

### Registro de imágenes (Amazon ECR)
3 repositorios: `tienda-frontend`, `tienda-backend`, `tienda-db`, con imágenes etiquetadas por versión/commit.

### Exposición pública
Un **Classic Load Balancer** (`tienda-clb`) recibe tráfico en el puerto 80 y lo reenvía al **NodePort 31585** del servicio `tienda-frontend`. Se usó Classic LB en vez de un Application Load Balancer administrado por el AWS Load Balancer Controller / EKS Auto Mode, debido a una restricción de permisos del entorno AWS Academy Learner Lab (ver sección de problemas resueltos).

---

## 🔄 Pipeline CI/CD (GitHub Actions)

Archivo: `.github/workflows/deploy-eks.yml`, disparado en cada `push` a `main`.

**Pasos del pipeline:**
1. Checkout del código
2. Configuración de credenciales AWS (vía GitHub Secrets)
3. Login a Amazon ECR
4. Generación de un tag único de imagen basado en el hash del commit
5. Build & push de las 3 imágenes (frontend, backend, db)
6. Instalación de `kubectl` y configuración del `kubeconfig` contra EKS
7. Aplicación de manifiestos base (namespace, secret y recursos de MySQL)
8. Actualización de backend y frontend (`kubectl set image` + `rollout status`)
9. Aplicación de los HPA
10. Verificación final (`kubectl get pods/svc/hpa`)

**GitHub Secrets requeridos:**
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`, `EKS_CLUSTER_NAME`, `EKS_NAMESPACE`

> ⚠️ **Limitación conocida:** las credenciales de AWS Academy Learner Lab expiran cada ~4 horas. Si el laboratorio se reinicia, hay que actualizar los GitHub Secrets antes de volver a ejecutar el pipeline.

---

## ☸️ Despliegue en Kubernetes

Namespace `tienda`, con:
- `mysql-secret.yaml`: contraseña de MySQL como **Secret de Kubernetes** (en vez de texto plano en el manifiesto)
- Deployments con **2 réplicas** (backend y frontend) y **1 réplica** (db)
- **Readiness y liveness probes** en los 3 componentes
- **Requests/limits** de CPU y memoria (necesarios para que el HPA funcione)
- **HPA:** backend escala 2–10 réplicas (umbral CPU 70%), frontend escala 2–6 réplicas (umbral CPU 60%)

### Verificar el despliegue
```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
```

---

## 🛠️ Problemas reales resueltos durante el desarrollo

Documentamos estos problemas porque reflejan el proceso real de configurar EKS en un entorno restringido (AWS Academy Learner Lab), y las decisiones técnicas tomadas para resolverlos:

1. **Node Group no se creaba:** las subredes públicas no tenían habilitada la auto-asignación de IP pública. Se corrigió habilitándola.

2. **Nodos en estado `NotReady` indefinidamente:** el clúster tenía activado parcialmente EKS Auto Mode, y la Access Entry de `LabRole` estaba configurada como tipo `EC2` en vez de `EC2_LINUX`. Se corrigió la Access Entry y se relanzaron las instancias.

3. **Add-ons de red faltantes:** el clúster no tenía instalados `vpc-cni`, `kube-proxy` ni `coredns`, por lo que los nodos nunca reportaban `Ready`. Se instalaron los 3 add-ons desde la consola de EKS.

4. **Imágenes mal etiquetadas en ECR:** por error, se subieron imágenes al repositorio equivocado. Se detectó revisando `kubectl logs` (mostraban un componente distinto al esperado) y se corrigió reconstruyendo con `--no-cache` y agregando `imagePullPolicy: Always` a los Deployments.

5. **AWS Load Balancer Controller no funcionaba:** el plan inicial era usar un Service tipo `LoadBalancer` con EKS Auto Mode para crear un ALB automáticamente. Falló con `AccessDenied: ... sts:TagSession on resource LabRole`, una restricción explícita del entorno AWS Academy. **Solución:** se cambió el Service de frontend a `NodePort` y se creó manualmente un Classic Load Balancer apuntando a ese puerto.

6. **Instancias "Fuera de servicio" en el Load Balancer:** las instancias del Node Group no tenían asociado el Security Group con el puerto del NodePort abierto. Se agregó la regla en el Security Group correcto.

7. **Metrics Server sin métricas (`<unknown>` en el HPA):** el add-on instalado inicialmente exigía un nodo gestionado por Karpenter (`nodepool=system`), incompatible con un Managed Node Group tradicional. Se reemplazó por el Metrics Server estándar de `kubernetes-sigs`, y se resolvió además un segundo problema de `Service` con selector desalineado a las labels del pod, que causaba `MissingEndpoints` en el APIService.

---

## ✅ Estado final verificado

- 5 pods `1/1 Running` (2 backend, 2 frontend, 1 base de datos)
- HPA reportando métricas reales de CPU
- Aplicación accesible públicamente vía el Classic Load Balancer
- CRUD completo probado y funcional (crear, leer, editar, eliminar productos), persistido en MySQL
