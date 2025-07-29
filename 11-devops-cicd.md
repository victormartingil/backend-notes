# DevOps & CI/CD - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### DevOps Culture
- **Definición**: Integración de desarrollo y operaciones para mejorar colaboración
- **Principios**: Automatización, colaboración, mejora continua, feedback rápido
- **Beneficios**: Deployments más frecuentes, menor tiempo de recuperación, mayor calidad

### CI/CD Pipeline
- **Continuous Integration**: Integración automática de código frecuente
- **Continuous Delivery**: Automatización hasta pre-producción
- **Continuous Deployment**: Automatización completa hasta producción

## 2. GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Set up JDK
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Cache Gradle dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*') }}
    
    - name: Run tests
      run: ./gradlew test
      env:
        SPRING_PROFILES_ACTIVE: test
        DATABASE_URL: jdbc:postgresql://localhost:5432/testdb
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./build/reports/jacoco/test/jacocoTestReport.xml

  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Build application
      run: ./gradlew bootJar
      
    - name: Build Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: myorg/my-app:latest
        
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment: production
    
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/
        kubectl rollout status deployment/my-app
```

## 3. Docker

### Multi-stage Dockerfile
```dockerfile
# Build stage
FROM gradle:8-jdk17 AS builder

WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src/ src/
RUN gradle bootJar --no-daemon

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache curl && \
    rm -rf /var/cache/apk/*

WORKDIR /app

# Copy built artifact
COPY --from=builder /app/build/libs/*.jar app.jar
RUN chown appuser:appgroup app.jar

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

# JVM optimization for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

## 4. Terraform

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "my-app/terraform.tfstate"
    region = "us-east-1"
  }
}

# VPC
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "${var.project_name}-cluster"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    main = {
      desired_size = 2
      max_size     = 10
      min_size     = 1
      
      instance_types = ["t3.medium"]
    }
  }
}

# RDS Database
resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-db"
  
  engine         = "postgres"
  engine_version = "13.7"
  instance_class = "db.t3.micro"
  
  allocated_storage = 20
  storage_encrypted = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  backup_retention_period = 7
  skip_final_snapshot = var.environment != "production"
}
```

## 5. Monitoring

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    static_configs:
      - targets: ['my-app:8080']
    metrics_path: '/actuator/prometheus'
    
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Alert Rules
```yaml
# alert_rules.yml
groups:
  - name: application
    rules:
      - alert: HighMemoryUsage
        expr: (java_lang_memory_used_bytes{area="heap"} / java_lang_memory_max_bytes{area="heap"}) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          
      - alert: ApplicationDown
        expr: up{job="spring-boot-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
```

## 6. Security en CI/CD

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'my-app'
        path: '.'
        
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        
    - name: Run CodeQL Analysis
      uses: github/codeql-action/init@v2
      with:
        languages: java
```

## 7. Preguntas Típicas

### ¿Cuál es la diferencia entre CI, CD y CD?
- **CI**: Integración automática de código
- **Continuous Delivery**: Automatización hasta pre-producción
- **Continuous Deployment**: Automatización completa hasta producción

### ¿Cómo implementar Blue-Green Deployment?
- **Concepto**: Dos entornos idénticos
- **Proceso**: Deploy en entorno inactivo, switch de tráfico
- **Ventajas**: Zero downtime, rollback instantáneo

### ¿Qué es Infrastructure as Code?
- **Definición**: Gestión de infraestructura mediante código
- **Herramientas**: Terraform, CloudFormation
- **Beneficios**: Versionado, reproducibilidad

### ¿Cómo manejar secrets?
- **Nunca en código**: Variables de entorno
- **Secret managers**: GitHub Secrets, Vault
- **Rotación**: Cambio periódico
- **Least privilege**: Mínimos permisos

## 8. Mejores Prácticas

### Pipeline Security
```yaml
jobs:
  security-check:
    runs-on: ubuntu-latest
    steps:
      # Pin action versions
      - uses: actions/checkout@v4.1.1
      
      # Verify dependencies
      - name: Verify dependencies
        run: gradle dependencies --verify-metadata
        
      # Vulnerability scan
      - name: Scan vulnerabilities
        run: |
          docker run --rm -v $(pwd):/workspace \
            aquasec/trivy fs --exit-code 1 /workspace
```

### Environment Promotion
```bash
#!/bin/bash
# promote.sh

SOURCE_ENV=${1:-staging}
TARGET_ENV=${2:-production}
IMAGE_TAG=${3:-latest}

echo "Promoting from $SOURCE_ENV to $TARGET_ENV"

# Run smoke tests
./scripts/smoke-tests.sh "$SOURCE_ENV"

# Deploy to target
helm upgrade --install "myapp-$TARGET_ENV" ./helm/myapp \
    --namespace "$TARGET_ENV" \
    --set image.tag="$IMAGE_TAG" \
    --wait

# Verify deployment
kubectl wait --for=condition=ready pod \
    -l app=myapp \
    -n "$TARGET_ENV" \
    --timeout=300s

echo "Promotion completed successfully"
```

## Puntos Clave para Recordar

1. **Automation**: Automatizar desde commit hasta producción
2. **Security**: Integrar security scans en cada etapa
3. **Testing**: Múltiples niveles (unit, integration, E2E)
4. **Monitoring**: Observabilidad desde día uno
5. **Rollback**: Capacidad de rollback rápido
6. **Infrastructure as Code**: Versionar infraestructura
7. **Secrets Management**: Nunca hardcodear secrets
8. **Environment Parity**: Entornos consistentes
9. **Blue-Green**: Deploy sin downtime
10. **Metrics**: Medir todo para mejora continua
