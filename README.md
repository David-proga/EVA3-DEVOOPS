#  Tienda de Alimentos para Perritos — Despliegue en Amazon EKS

## Descripción del Proyecto

Aplicación web CRUD para la gestión de productos de una tienda de alimentos para mascotas, desplegada en un clúster de **Amazon EKS (Elastic Kubernetes Service)** como parte de la Evaluación Parcial N°3 del curso **ISY1101 — Introducción a Herramientas DevOps** en DUOC UC.

### Contexto: Innovatech Chile

La empresa **Innovatech Chile**, tras completar la etapa de contenedorización (EP2) y montar infraestructura base en AWS (EP1), avanza hacia la **automatización y orquestación productiva**. El equipo técnico debe crear un entorno de orquestación escalable, tolerante a fallos y automatizable, garantizando despliegues continuos desde GitHub mediante GitHub Actions.

---

## Arquitectura del Sistema

```
┌──────────────┐       ┌──────────────────────────────────────────────────────────┐
│   Developer  │       │                    Amazon EKS Cluster                    │
│              │       │                   (innovatech-eks)                       │
│  git push    │       │                                                          │
│  to main     │       │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐   │
└──────┬───────┘       │  │  Frontend    │  │   Backend    │  │    MySQL DB   │   │
       │               │  │  (nginx)     │──│  (Node.js)   │──│   (mysql:8.0) │   │
       ▼               │  │  Port: 80    │  │  Port: 3001  │  │   Port: 3306  │   │
┌──────────────┐       │  │  Replicas: 2 │  │  Replicas: 2 │  │   Replicas: 1 │   │
│   GitHub     │       │  └─────────────┘  └──────────────┘  └───────────────┘   │
│   Actions    │       │                                                          │
│              │       │  ┌─────────────────────────────────────────────────────┐  │
│  build ──►   │       │  │  HPA (Horizontal Pod Autoscaler)                   │  │
│  push  ──►   │──────►│  │  • Backend:  min 2, max 10, target CPU 70%         │  │
│  deploy ──►  │       │  │  • Frontend: min 2, max 6,  target CPU 60%         │  │
└──────────────┘       │  └─────────────────────────────────────────────────────┘  │
       │               └──────────────────────────────────────────────────────────┘
       ▼
┌──────────────┐
│  Amazon ECR  │
│  (Imágenes)  │
│  • frontend  │
│  • backend   │
│  • db        │
└──────────────┘
```

---

## Stack Tecnológico

| Componente | Tecnología |
|------------|-----------|
| **Frontend** | HTML + JavaScript + Nginx Alpine |
| **Backend** | Node.js + Express + mysql2 |
| **Base de Datos** | MySQL 8.0 |
| **Contenedores** | Docker |
| **Orquestador** | Amazon EKS (Kubernetes 1.35) |
| **Registro de Imágenes** | Amazon ECR |
| **CI/CD** | GitHub Actions |
| **Autoscaling** | Horizontal Pod Autoscaler (HPA) |
| **Infraestructura** | AWS (VPC, EC2, IAM, EKS) |

---

## Estructura del Repositorio

```
EVA3-DEVOOPS/
├── .github/
│   └── workflows/
│       └── deploy-eks.yml          # Pipeline CI/CD de GitHub Actions
├── frontend/
│   ├── Dockerfile                  # Imagen nginx con app estática
│   ├── default.conf                # Config nginx con proxy al backend
│   ├── index.html                  # Página principal
│   └── app.js                      # Lógica CRUD del frontend
├── backend/
│   ├── Dockerfile                  # Imagen Node.js
│   ├── server.js                   # API REST Express + MySQL
│   └── package.json                # Dependencias
├── db/
│   └── Dockerfile                  # Imagen base de datos
├── k8s/
│   ├── namespace.yaml              # Namespace 'tienda'
│   ├── mysql-secret.yaml           # Credenciales MySQL (Secret)
│   ├── mysql-deployment.yaml       # Deployment MySQL
│   ├── mysql-service.yaml          # Service MySQL (ClusterIP)
│   ├── backend-deployment.yaml     # Deployment Backend
│   ├── backend-service.yaml        # Service Backend (NodePort)
│   ├── frontend-deployment.yaml    # Deployment Frontend
│   ├── frontend-service.yaml       # Service Frontend (NodePort)
│   ├── backend-hpa.yaml            # HPA Backend
│   └── frontend-hpa.yaml          # HPA Frontend
└── README.md                       # Este archivo
```

---

## Requisitos Previos

- **Cuenta AWS Academy** con Learner Lab activo
- **AWS CLI v2** instalado y configurado
- **kubectl** instalado
- **Docker Desktop** instalado
- **Git** instalado
- **Visual Studio Code** (recomendado)

---

## Configuración del Clúster EKS

### 1. Crear el Clúster

El clúster fue creado en la consola de AWS con la siguiente configuración:

| Parámetro | Valor |
|-----------|-------|
| **Nombre** | `innovatech-eks` |
| **Versión Kubernetes** | 1.35 |
| **Rol IAM** | `LabRole` |
| **VPC** | Default VPC |
| **Subnets** | 5 subnets públicas en distintas AZ |
| **Endpoint** | Público |

### 2. Crear el Node Group

| Parámetro | Valor |
|-----------|-------|
| **Nombre** | `nodegroup-tienda` |
| **Tipo de instancia** | `t3.medium` |
| **AMI** | Amazon Linux 2023 |
| **Nodos deseados** | 2 |
| **Nodos mínimos** | 2 |
| **Nodos máximos** | 3 |
| **Disco** | 20 GiB |

### 3. Add-ons Instalados

Los siguientes add-ons son necesarios para el correcto funcionamiento del clúster:

- **vpc-cni**: Plugin de red para asignar IPs a los pods dentro de la VPC
- **kube-proxy**: Proxy de red de Kubernetes
- **coredns**: Servidor DNS interno del clúster
- **metrics-server**: Recolección de métricas de CPU/memoria para el HPA

### 4. Access Entry (Autenticación API Mode)

El clúster usa el modo de autenticación `API`. Se configuró un Access Entry tipo `EC2_LINUX` para el rol `LabRole`, permitiendo que los nodos se unan al clúster.

```bash
aws eks create-access-entry --cluster-name innovatech-eks \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/LabRole \
  --type EC2_LINUX
```

### 5. Conectar kubectl al Clúster

```bash
aws eks update-kubeconfig --region us-east-1 --name innovatech-eks
kubectl get nodes  # Verificar nodos en estado Ready
```

---

## Repositorios en Amazon ECR

Se crearon 3 repositorios privados en Amazon ECR para almacenar las imágenes Docker:

| Repositorio | Descripción |
|-------------|-------------|
| `tienda-frontend` | Imagen del frontend (nginx + archivos estáticos) |
| `tienda-backend` | Imagen del backend (Node.js + Express) |
| `tienda-db` | Imagen de la base de datos |

---

## Pipeline CI/CD (GitHub Actions)

### Flujo del Pipeline

El archivo `.github/workflows/deploy-eks.yml` define el pipeline completo:

1. **Checkout**: Descarga el código del repositorio
2. **Configurar AWS**: Autenticación con credenciales del Learner Lab
3. **Login ECR**: Autenticación en el registro de imágenes
4. **Build & Push**: Construcción de imágenes Docker y publicación en ECR
   - Frontend
   - Backend
   - Base de datos
5. **Instalar kubectl**: Herramienta de línea de comandos para Kubernetes
6. **Configurar kubeconfig**: Conexión al clúster EKS
7. **Aplicar manifests**: Despliegue de recursos en Kubernetes
   - Namespace
   - Secrets de MySQL
   - Deployment y Service de MySQL
   - Deployment y Service del Backend
   - Deployment y Service del Frontend
8. **Aplicar HPA**: Configuración de autoscaling
9. **Verificación**: Estado final de pods y servicios

### Ejecución del Pipeline

El pipeline se ejecuta automáticamente con cada `push` a la rama `main`, o manualmente desde la pestaña **Actions** en GitHub.

### Secrets Configurados en GitHub

| Secret | Descripción |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Credencial de acceso AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión AWS Academy |
| `AWS_REGION` | Región de AWS (`us-east-1`) |
| `EKS_CLUSTER_NAME` | Nombre del clúster (`innovatech-eks`) |
| `EKS_NAMESPACE` | Namespace de Kubernetes (`tienda`) |

> **Importante**: Las credenciales de AWS Academy expiran cada ~4 horas. Es necesario actualizar `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN` cada vez que se reinicia el Learner Lab.

---

## Despliegue en Kubernetes

### Servicios Desplegados

| Servicio | Tipo | Puerto | Acceso |
|----------|------|--------|--------|
| `tienda-frontend` | NodePort | 80 → 30080 | Público vía IP del nodo |
| `tienda-backend` | NodePort | 3001 → 30081 | Interno/Público |
| `tienda-db` | ClusterIP | 3306 | Solo interno del clúster |

### Comunicación entre Servicios

- **Usuario** → `http://<NODE_IP>:30080` → **Frontend (nginx)**
- **Frontend** → proxy inverso (`/api/`) → **Backend** (`http://tienda-backend:3001`)
- **Backend** → conexión MySQL → **Base de datos** (`tienda-db:3306`)

### Verificar el Despliegue

```bash
# Ver pods en ejecución
kubectl get pods -n tienda

# Ver servicios
kubectl get svc -n tienda

# Ver HPA
kubectl get hpa -n tienda

# Ver métricas de pods
kubectl top pods -n tienda

# Ver métricas de nodos
kubectl top nodes

# Ver logs del backend
kubectl logs deployment/tienda-backend -n tienda --tail=20

# Ver logs del frontend
kubectl logs deployment/tienda-frontend -n tienda --tail=20
```

### Acceder a la Aplicación

1. Obtener la IP pública de un nodo:
   ```bash
   kubectl get nodes -o wide
   ```
2. Abrir en el navegador: `http://<EXTERNAL-IP>:30080`

---

## Autoscaling (HPA)

Se configuró Horizontal Pod Autoscaler para el frontend y backend:

| Deployment | Métrica | Umbral | Min Pods | Max Pods |
|------------|---------|--------|----------|----------|
| `tienda-backend` | CPU | 70% | 2 | 10 |
| `tienda-frontend` | CPU | 60% | 2 | 6 |

### Justificación de los Umbrales

- **Backend (70% CPU)**: Se eligió este umbral para permitir que el backend maneje picos de consultas a la base de datos sin saturarse. Con 2 réplicas mínimas se garantiza alta disponibilidad; el máximo de 10 permite absorber carga alta.

- **Frontend (60% CPU)**: Un umbral más bajo que el backend porque el frontend debe responder rápidamente a las peticiones de los usuarios. El mínimo de 2 réplicas asegura disponibilidad ante la caída de un nodo.

### Verificar Autoscaling

```bash
# Ver estado del HPA
kubectl get hpa -n tienda

# Detalle del HPA
kubectl describe hpa tienda-backend-hpa -n tienda
```

---

## Gestión de Credenciales y Seguridad

- Las credenciales de AWS se almacenan como **GitHub Secrets**, nunca en el código fuente
- La contraseña de MySQL se almacena como **Kubernetes Secret** (`mysql-secret`)
- Los Secrets de GitHub ocultan automáticamente los valores en los logs del pipeline
- Las credenciales de AWS Academy se renuevan cada sesión del Learner Lab

---

## Problemas Encontrados y Soluciones

### 1. NodeCreationFailure — Nodos no se unían al clúster
**Causa**: El clúster usaba autenticación modo `API` en vez de `CONFIG_MAP`, por lo que el ConfigMap `aws-auth` no tenía efecto.
**Solución**: Crear un Access Entry tipo `EC2_LINUX` para el rol `LabRole`.

### 2. Pods NotReady — CNI plugin not initialized
**Causa**: Los add-ons de red (vpc-cni, kube-proxy, coredns) no estaban instalados.
**Solución**: Instalar los add-ons manualmente con `aws eks create-addon`.

### 3. ImagePullBackOff en tienda-db
**Causa**: El deployment apuntaba a una imagen ECR de otra cuenta AWS.
**Solución**: Cambiar la imagen a `mysql:8.0` (imagen oficial de Docker Hub).

### 4. LoadBalancer en estado Pending
**Causa**: Las subnets no tenían el tag `kubernetes.io/role/elb` y el AWS Load Balancer Controller no estaba instalado.
**Solución**: Cambiar el tipo de servicio a `NodePort` para acceso directo por IP del nodo.

### 5. 504 Gateway Timeout en Frontend
**Causa**: Nginx no resolvía el DNS interno del backend tras reinicio de servicios.
**Solución**: Reiniciar el deployment del frontend con `kubectl rollout restart`.

### 6. Tabla "productos" no existe
**Causa**: La imagen oficial de MySQL no crea tablas automáticamente, solo la base de datos.
**Solución**: Crear la tabla manualmente con `kubectl exec` y agregar la variable `MYSQL_DATABASE` al deployment.

### 7. Metrics Server no arrancaba
**Causa**: El deployment tenía un `nodeSelector` que requería nodos con label `karpenter.sh/nodepool=system`.
**Solución**: Eliminar el `nodeSelector` del deployment con `kubectl edit`.

---

## Cómo Replicar el Proyecto

1. **Clonar el repositorio**:
   ```bash
   git clone https://github.com/David-proga/EVA3-DEVOOPS.git
   cd EVA3-DEVOOPS
   ```

2. **Crear el clúster EKS** siguiendo la sección "Configuración del Clúster EKS"

3. **Crear repositorios ECR**: `tienda-frontend`, `tienda-backend`, `tienda-db`

4. **Configurar Secrets en GitHub** con las credenciales de AWS Academy

5. **Hacer push a main** para activar el pipeline:
   ```bash
   git add .
   git commit -m "deploy: despliegue inicial"
   git push origin main
   ```

6. **Crear la tabla de productos** en MySQL:
   ```bash
   kubectl exec -it deployment/tienda-db -n tienda -- mysql -u root -padmin123 tienda_perritos \
     -e "CREATE TABLE IF NOT EXISTS productos (id INT AUTO_INCREMENT PRIMARY KEY, nombre VARCHAR(255) NOT NULL, descripcion TEXT, precio DECIMAL(10,2) NOT NULL, stock INT NOT NULL);"
   ```

7. **Acceder a la aplicación** por el NodePort del frontend

---

## Autores

| Nombre | GitHub |
|--------|--------|
| Fabrizzio Luciano Mego Delgado | [@F-brizzio](https://github.com/F-brizzio) |
| David maximiliano sereño flores | [@David-proga](https://github.com/David-proga) |

---

## Asignatura

**ISY1101 — Introducción a Herramientas DevOps**
DUOC UC — Escuela de Informática y Telecomunicaciones
Evaluación Parcial N°3 — Semestre 2025
