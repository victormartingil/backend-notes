# ☁️ AWS para Arquitectos Backend

Amazon Web Services (AWS) es la nube líder en infraestructura escalable, resiliente y global. Como arquitecto backend o consultor, dominar los servicios clave y su diseño es fundamental para construir soluciones profesionales.

---

## 🎯 Objetivo

* Diseñar sistemas backend resilientes, seguros y económicos en AWS
* Comprender los servicios base y patrones arquitectónicos comunes
* Alinear arquitectura cloud con negocio, seguridad y costes

---

## 🔧 Servicios esenciales para backend

### 🔹 Compute

* **EC2**: instancias virtuales personalizables
* **ECS / EKS**: contenedores con orquestación (Docker / Kubernetes)
* **Lambda**: funciones serverless para eventos o tareas específicas

### 🔹 Networking

* **VPC**: red virtual privada
* **Security Groups**: cortafuegos por servicio
* **API Gateway / Load Balancer**: exposición controlada de servicios

### 🔹 Bases de datos

* **RDS** (PostgreSQL, MySQL): relacional gestionada
* **DynamoDB**: NoSQL serverless
* **Aurora**: relacional escalable, compatible con MySQL/PostgreSQL

### 🔹 Mensajería y eventos

* **SQS**: colas (asíncrono, desacoplado)
* **SNS**: publicación/suscripción (push notification, fan-out)
* **EventBridge**: eventos internos o integraciones SaaS

### 🔹 Storage

* **S3**: almacenamiento de objetos (documentos, backups, estáticos)
* **EFS**: sistema de archivos distribuido (multiinstancia)

---

## 🛠️ Infraestructura como Código

* **Terraform**: herramienta declarativa multiplataforma
* **AWS CDK**: infraestructura con código (TypeScript, Java, Python...)
* Uso de módulos reutilizables, variables, entornos

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-logs-bucket"
  acl    = "private"
}
```

---

## 💵 Optimización de Costes

* Uso de instancias reservadas o spot si aplica
* Dimensionamiento adecuado (CPU/RAM)
* Monitorización de gasto con **Cost Explorer**, **Budgets**, **Tags**
* Evitar servicios en ejecución innecesarios (EC2, RDS)

---

## 🔐 Seguridad

* **IAM**: roles y permisos mínimos necesarios (least privilege)
* **Secrets Manager / Parameter Store**: evitar secretos en código
* **MFA**, alertas y revisión de logs CloudTrail
* Cifrado en tránsito y en reposo (TLS, KMS)

---

## ✅ Qué se espera de ti como consultor

* Diseñar arquitecturas cloud seguras, mantenibles y con costes razonables
* Saber elegir entre EC2, EKS o Lambda según caso de uso
* Automatizar despliegues con Terraform, GitHub Actions, etc.
* Identificar puntos de fallo, overengineering o sobrecoste
* Justificar las decisiones con foco en negocio (SLA, disponibilidad, velocidad, coste)

---

## 🧪 Ejercicio para practicar

1. Crear una arquitectura simple: API REST en Spring Boot desplegada en EKS con RDS + S3
2. Gestionar la infraestructura con Terraform
3. Añadir SQS para desacoplar procesamiento
4. Medir coste mensual estimado y justificar elección

---

## 📚 Recursos

* AWS Well-Architected Framework: [aws.amazon.com/architecture](https://aws.amazon.com/architecture/well-architected/)
* Terraform: [terraform.io](https://developer.hashicorp.com/terraform)
* CDK: [aws.amazon.com/cdk](https://aws.amazon.com/cdk/)
* Cursos recomendados: "AWS para desarrolladores", "EKS + Spring Boot"

---
