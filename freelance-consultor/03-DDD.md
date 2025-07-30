# 🧠 Domain-Driven Design (DDD)

El Domain-Driven Design (DDD) es una metodología de diseño de software centrada en el **modelo del dominio** y en **colaborar con expertos del negocio** para construir software alineado a la realidad.

---

## 🎯 Objetivo de DDD

* Modelar el dominio con precisión y lenguaje compartido
* Aislar la lógica de negocio de detalles técnicos
* Dividir sistemas complejos en contextos bien definidos

---

## 🧩 Conceptos Estratégicos

### 🔹 Ubiquitous Language

* Usar un **lenguaje común** entre técnicos y negocio
* Reflejar este lenguaje en el código: nombres, entidades, operaciones

```kotlin
// No usar "User" si negocio habla de "Customer"
data class Customer(val id: UUID, val fullName: String)
```

### 🔹 Bounded Context

* Frontera lógica donde un modelo tiene sentido y coherencia
* Dentro de un bounded context, el lenguaje es consistente

```text
Ejemplo: "Billing" y "CRM" pueden tener ambos un objeto llamado "Customer", pero con datos y reglas diferentes.
```

### 🔹 Context Map

* Relación entre los distintos bounded contexts
* Incluye patrones como: Partnership, Shared Kernel, Anti-corruption Layer (ACL), etc.

---

## 🧩 Conceptos Tácticos

### 🔸 Entidades (Entities)

* Tienen identidad persistente
* Cambian a lo largo del tiempo

```kotlin
data class Order(val id: OrderId, val items: List<OrderItem>)
```

### 🔸 Value Objects (VO)

* Sin identidad, definidos solo por su valor
* Inmutables

```kotlin
data class Money(val amount: BigDecimal, val currency: String)
```

### 🔸 Aggregates & Aggregate Root

* Unidad de consistencia y transacción
* El `AggregateRoot` es la única entrada pública para modificar el estado

```kotlin
class Order(val id: OrderId, val items: List<OrderItem>) {
  fun addItem(item: OrderItem) { ... }
  fun totalAmount(): Money { ... }
}
```

### 🔸 Domain Services

* Representan operaciones del dominio que no encajan en una entidad o VO

```kotlin
interface CurrencyConverter {
  fun convert(money: Money, toCurrency: String): Money
}
```

### 🔸 Repositories

* Abstracción para recuperar y persistir aggregates
* Deben respetar el lenguaje del dominio

```kotlin
interface OrderRepository {
  fun findById(id: OrderId): Order?
  fun save(order: Order)
}
```

---

## ✅ Qué se espera de ti como consultor

* Saber identificar entidades, VOs y servicios de dominio en una conversación con negocio
* Proponer estructuras de dominio que representen correctamente reglas y restricciones
* Ser capaz de mapear bounded contexts y sus relaciones
* Detectar acoplamientos indebidos y proponer estrategias (ACL, traducción de modelos…)
* Promover el uso del lenguaje ubicuo en todo el código

---

## 💬 Ventajas clave

* Mejora comunicación entre equipos técnicos y negocio
* Software alineado con reglas reales y futuras del negocio
* Evita acoplamientos innecesarios entre módulos o contextos

---

## 🧪 Ejercicio para practicar

1. Modela un dominio sencillo: alquiler de coches (`CarRental`)
2. Define entidades: `Rental`, `Customer`, `Car`
3. Crea value objects: `LicensePlate`, `Money`, `RentalPeriod`
4. Limita operaciones al `AggregateRoot`
5. Crea interfaz de repositorio + un servicio de dominio

---

## 📚 Recursos

* Libro: *Implementing Domain-Driven Design* – Vaughn Vernon
* Libro: *Domain-Driven Design* – Eric Evans
* Curso gratuito: [DDD Quickly (InfoQ)](https://www.infoq.com/minibooks/domain-driven-design-quickly/)
* Ejemplo Kotlin: [ddd-by-examples](https://github.com/ddd-by-examples/library)

---
