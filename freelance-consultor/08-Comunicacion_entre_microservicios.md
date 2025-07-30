# 🔄 Comunicación entre Servicios (Sync/Async)

Diseñar correctamente la comunicación entre microservicios es esencial para la resiliencia, la escalabilidad y el mantenimiento de un sistema distribuido.

---

## 🎯 Objetivo

* Permitir integración clara entre Bounded Contexts
* Asegurar bajo acoplamiento y alta disponibilidad
* Elegir entre sincronía/asíncronía según el escenario de negocio

---

## 📡 Tipos de comunicación

### 🔹 Sincrónica (REST/gRPC)

* Respuesta inmediata (cliente espera)
* Útil para queries simples o flujos transaccionales acotados
* Protocolos: HTTP REST, gRPC

```kotlin
// Ejemplo REST usando Spring Web
@PostMapping("/payments")
fun process(@RequestBody req: PaymentRequest): ResponseEntity<*> {
  return paymentService.charge(req)
}
```

**Consideraciones**:

* Sensible a fallos de red o tiempo de espera
* Necesita circuit breakers o timeouts bien configurados

---

### 🔹 Asíncrona (Kafka, RabbitMQ, SNS/SQS)

* Productor publica eventos, el consumidor procesa más tarde
* Ideal para desacoplar servicios, escalar o integrar procesos lentos

```kotlin
// Evento OrderCreated emitido por el servicio de pedidos
{
  "eventType": "OrderCreated",
  "orderId": "123",
  "amount": 45.00
}
```

**Consideraciones**:

* Se necesita estrategia de reintento, idempotencia, deduplicación
* Uso del patrón Outbox + CDC para consistencia eventual

---

## 🧰 Patrones clave

### 🔸 Outbox Pattern

* Publicar eventos como parte de la transacción DB, y emitirlos luego de forma segura

### 🔸 Event-Carried State Transfer

* Incluir datos relevantes en el evento para que otros servicios no necesiten llamar de nuevo al origen

### 🔸 Idempotencia

* Procesar el mismo evento varias veces no debe generar inconsistencias

### 🔸 Anti-Corruption Layer (ACL)

* Adaptar o traducir el modelo externo para proteger tu dominio

---

## ✅ Qué se espera de ti como consultor

* Decidir correctamente entre REST y eventos según el tipo de interacción
* Diseñar eventos ricos, versionables y autoexplicativos
* Aplicar contratos API y contract testing (OpenAPI, Pact)
* Asegurar que los consumidores no dependan de llamadas en cadena
* Introducir patrones como outbox o ACL cuando sea necesario

---

## 🧪 Ejercicio para practicar

1. Implementa un servicio `orders` que expone `/orders` y publica un evento `OrderCreated`
2. Crea otro servicio `billing` que escucha `OrderCreated` y genera una factura
3. Usa Testcontainers con Kafka embebido para probar la comunicación
4. Añade un retry automático con backoff para el consumer

---

## 📚 Recursos

* Chris Richardson – [Microservices Communication Styles](https://microservices.io/patterns/index.html)
* Spring Cloud Stream – [spring.io](https://spring.io/projects/spring-cloud-stream)
* Artículo: *Reliable Messaging with the Outbox Pattern* – Martin Fowler

---
