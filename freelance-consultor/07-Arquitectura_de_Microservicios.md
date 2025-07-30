# 🧩 Arquitectura de Microservicios

La arquitectura de microservicios consiste en dividir una aplicación grande en **servicios pequeños e independientes** que se comunican entre sí. Cada servicio es autónomo, despliega por separado y está alineado con un contexto de negocio.

---

## 🎯 Objetivo

* Escalabilidad independiente por dominio
* Equipos pequeños, autónomos y dueños de su servicio
* Despliegue continuo sin afectar a toda la plataforma

---

## 🧱 Principios clave

### 🔹 Bounded Context = 1 Microservicio

* Cada microservicio debe tener **una única responsabilidad** de negocio

### 🔹 Base de datos por servicio

* No compartir bases de datos entre servicios
* Cada uno gestiona su persistencia

### 🔹 Comunicación explícita

* HTTP (REST), Mensajes (Kafka, RabbitMQ), gRPC
* Contratos claros, versionados y testeables

### 🔹 Independencia total

* Cada servicio se construye, prueba y despliega solo
* Independencia tecnológica (si aplica)

---

## 🔁 Patrones importantes

### 🔸 API Gateway

* Entrada única a los servicios (routing, auth, throttling)

### 🔸 Service Discovery

* Servicios se registran y descubren dinámicamente (Eureka, Consul)

### 🔸 Circuit Breaker

* Tolerancia a fallos entre servicios (Resilience4j, Hystrix)

### 🔸 Config Server

* Config centralizada y versionada (Spring Cloud Config)

### 🔸 Event-Driven Architecture

* Comunicación asíncrona mediante eventos (Kafka, SNS/SQS)

---

## ✅ Qué se espera de ti como consultor

* Saber cuándo **NO** usar microservicios (equipo pequeño, app simple)
* Diseñar servicios alineados a Bounded Contexts con bajo acoplamiento
* Definir contratos API/eventos y estrategias de versionado
* Detectar smells: servicios con demasiadas responsabilidades, DBs compartidas, acoplamientos cíclicos
* Promover prácticas de observabilidad, testing distribuido y resiliencia

---

## 💬 Buenas prácticas

* Diseñar eventos ricos en información (event-carried state transfer)
* Usar outbox pattern para publicar eventos de forma transaccional
* Versionar APIs (ej: `/v1/orders`), evitar cambios incompatibles
* Documentar con OpenAPI + ejemplos
* Priorizar idempotencia en endpoints y consumers

---

## 🧪 Ejercicio para practicar

1. Define dos microservicios: `orders` y `billing`
2. `orders` expone una API para crear pedidos y emite un evento `OrderCreated`
3. `billing` escucha el evento y genera una factura
4. Usa base de datos separada por microservicio (in-memory o docker)
5. Añade trazabilidad simple (logs con `traceId`)

---

## 📚 Recursos

* Libro: *Building Microservices* – Sam Newman
* Guía oficial Spring Cloud: [spring.io/projects/spring-cloud](https://spring.io/projects/spring-cloud)
* Blog: *Microservices.io* – Chris Richardson

---
