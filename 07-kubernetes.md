# Kubernetes - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### Arquitectura de Kubernetes
```
┌─────────────────────────────────────────────────────────────────┐
│                        Master Node                              │
│  ┌─────────────┐ ┌──────────────┐ ┌────────────┐ ┌──────────────┐ │
│  │ API Server  │ │   etcd       │ │ Scheduler  │ │ Controller   │ │
│  │             │ │              │ │            │ │ Manager      │ │
│  └─────────────┘ └──────────────┘ └────────────┘ └──────────────┘ │
└─────────────────────────────────────────────────────────────────┘
           │                                │
           ▼                                ▼
┌─────────────────────┐            ┌─────────────────────┐
│    Worker Node 1    │            │    Worker Node 2    │
│ ┌─────────────────┐ │            │ ┌─────────────────┐ │
│ │     kubelet     │ │            │ │     kubelet     │ │
│ ├─────────────────┤ │            │ ├─────────────────┤ │
│ │   kube-proxy    │ │            │ │   kube-proxy    │ │
│ ├─────────────────┤ │            │ ├─────────────────┤ │
│ │ Container       │ │            │ │ Container       │ │
│ │ Runtime (Docker)│ │            │ │ Runtime (Docker)│ │
│ └─────────────────┘ │            │ └─────────────────┘ │
└─────────────────────┘            └─────────────────────┘
```

## 2. Objetos Básicos

### Pods
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
  - name: app-container
    image: my-app:1.0.0
    ports:
    - containerPort: 8080
    env:
    - name: SPRING_PROFILES_ACTIVE
      value: "production"
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Deployments
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

### Services
```yaml
# ClusterIP Service (interno)
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

---
# LoadBalancer Service (externo)
apiVersion: v1
kind: Service
metadata:
  name: my-app-external
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

## 3. ConfigMaps y Secrets

### ConfigMaps
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/myapp
    logging:
      level:
        com.example: INFO
```

### Secrets
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # Base64 encoded values
  database-username: cG9zdGdyZXM=  # postgres
  database-password: cGFzc3dvcmQ=  # password
```

## 4. Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 5. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

## 6. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre Deployment y StatefulSet?
- **Deployment**: Para aplicaciones stateless, pods intercambiables
- **StatefulSet**: Para aplicaciones stateful, pods con identidad única
- **Uso**: Deployment para APIs, StatefulSet para bases de datos

### ¿Qué es un Service y qué tipos existen?
- **ClusterIP**: Acceso interno al cluster
- **NodePort**: Expone puerto en todos los nodos
- **LoadBalancer**: Balanceador de carga externo
- **ExternalName**: Alias DNS para servicios externos

### ¿Cómo funciona el DNS en Kubernetes?
- **Formato**: `service-name.namespace.svc.cluster.local`
- **Resolución**: CoreDNS resuelve nombres de servicios
- **Ejemplo**: `postgres.default.svc.cluster.local`

### ¿Qué es un Ingress?
- **Definición**: Proxy reverso para enrutar tráfico HTTP/HTTPS
- **Funciones**: SSL termination, virtual hosting, path-based routing
- **Controllers**: nginx, traefik, HAProxy, AWS ALB

## 7. Comandos Útiles de kubectl

### Operaciones Básicas
```bash
# Ver pods
kubectl get pods
kubectl get pods -o wide

# Describir recursos
kubectl describe pod my-pod
kubectl describe service my-service

# Ver logs
kubectl logs my-pod
kubectl logs -f my-pod  # Follow logs

# Ejecutar comandos en pods
kubectl exec -it my-pod -- /bin/bash

# Port forwarding
kubectl port-forward pod/my-pod 8080:8080

# Aplicar manifiestos
kubectl apply -f deployment.yaml

# Scaling
kubectl scale deployment my-app --replicas=5

# Rolling updates
kubectl set image deployment/my-app container=my-app:v2
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app
```

## Puntos Clave para Recordar

1. **Pods**: Unidad mínima de deployment, contienen uno o más contenedores
2. **Services**: Abstracción para acceder a pods, tipos: ClusterIP, NodePort, LoadBalancer
3. **Deployments**: Gestionan ReplicaSets para aplicaciones stateless
4. **ConfigMaps/Secrets**: Gestión de configuración y datos sensibles
5. **Ingress**: Proxy reverso para tráfico HTTP/HTTPS externo
6. **Namespaces**: Aislamiento lógico de recursos
7. **HPA**: Autoscaling horizontal basado en métricas
8. **RBAC**: Control de acceso basado en roles
9. **Helm**: Gestor de paquetes para Kubernetes
10. **kubectl**: Herramienta CLI principal para interactuar con el cluster
