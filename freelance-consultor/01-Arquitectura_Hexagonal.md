# 🛡️ Arquitectura Hexagonal (Ports & Adapters)

La Arquitectura Hexagonal (también conocida como Ports & Adapters) es una forma de estructurar el software que **desacopla la lógica del negocio de la infraestructura**, permitiendo testabilidad, mantenibilidad y evolución sin fricción.

---

## 🌟 Objetivo de este patrón

Separar claramente:

* **Dominio** (la lógica de negocio pura)
* **Aplicación** (los casos de uso)
* **Infraestructura** (bases de datos, APIs externas, Kafka, web, etc.)

---

## 🧹 Estructura típica

```
           +----------------------------+
           |        Infraestructura     |
           | (Web, DB, Kafka, APIs...)  |
           +-------------+--------------+
                         |
                +--------v--------+
                |     Aplicación  |
                |  (Casos de uso) |
                +--------+--------+
                         |
               +---------v---------+
               |      Dominio      |
               | (Entities, VO, etc)|
               +-------------------+
```

---

## 🛠️ Componentes Clave

### 🔹 Dominio

* **Independiente del framework**
* Contiene: `Entity`, `ValueObject`, `DomainService`
* No tiene dependencias de Spring ni de persistencia

```kotlin
data class Customer(val id: UUID, val name: String)
```

---

### 🔹 Puerto (Port)

* Es una **interfaz** que define la comunicación desde el dominio hacia fuera
* Se define en el dominio o capa de aplicación

```kotlin
interface CustomerRepository {
    fun findById(id: UUID): Customer?
    fun save(customer: Customer)
}
```

---

### 🔹 Adaptador (Adapter)

* Es una **implementación concreta** de un puerto: DB, API REST, Kafka, Email…
* Vive en infraestructura

```kotlin
@Repository
class PostgresCustomerRepository(
  private val jpaRepo: CustomerJpaRepository
) : CustomerRepository {
  override fun findById(id: UUID): Customer? {
    return jpaRepo.findById(id).orElse(null)?.toDomain()
  }

  override fun save(customer: Customer) {
    jpaRepo.save(customer.toEntity())
  }
}
```

---

### 🔹 Caso de uso (Application Service)

* Orquesta entidades del dominio
* Coordina puertos y lógica
* No depende de adaptadores

```kotlin
class RegisterCustomerUseCase(
  private val repository: CustomerRepository
) {
  fun execute(command: RegisterCustomerCommand) {
    val customer = Customer(UUID.randomUUID(), command.name)
    repository.save(customer)
  }
}
```

---

### 🔹 Adaptador de entrada (Controller, Consumer)

* Invoca casos de uso desde el exterior (REST, Kafka…)
* Se considera parte de infraestructura

```kotlin
@RestController
@RequestMapping("/customers")
class CustomerController(
  private val registerCustomerUseCase: RegisterCustomerUseCase
) {
  @PostMapping
  fun register(@RequestBody request: RegisterCustomerRequest) {
    registerCustomerUseCase.execute(RegisterCustomerCommand(request.name))
  }
}
```

---

## ✅ Qué se espera de ti como consultor

* Saber **diseñar servicios** siguiendo esta separación de capas
* Detectar cuándo la lógica de negocio está “colándose” en infraestructura
* Ser capaz de **extraer puertos** para permitir testabilidad
* Aplicar este patrón incluso en microservicios pequeños
* Explicar las ventajas a equipos menos experimentados
* Promover nombres claros: `*Port`, `*Adapter`, `*UseCase`, `*Command`

---

## 💬 Ventajas clave

* Alta testabilidad (el dominio no depende de Spring ni frameworks)
* Permite cambiar adaptadores sin tocar la lógica
* Limpia separación de responsabilidades
* Facilidad para hacer tests de integración y contract testing
* Promueve foco en el **comportamiento del sistema**, no en su tecnología

---

## 🧪 Ejercicio para practicar

1. Crea un servicio `OrderService` con un caso de uso `PlaceOrderUseCase`
2. Define un puerto `OrderRepository` y su implementación `InMemoryOrderRepository`
3. Usa un controller REST como entrada
4. Añade un test unitario del caso de uso sin Spring

---

## 📚 Recursos

* [Hexagonal Architecture by Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
* [Vídeo de ejemplo en Kotlin + Spring Boot (YouTube)](https://www.youtube.com/watch?v=tEHuHtehQkY)
* Libro: *Clean Architecture* – Robert C. Martin

---
