# 🧼 Clean Architecture

La Clean Architecture, popularizada por Robert C. Martin (Uncle Bob), es un enfoque de diseño que busca **separar las distintas capas de una aplicación**, manteniendo la lógica del negocio independiente de frameworks, bases de datos, UI o agentes externos.

---

## 🎯 Objetivo de este patrón

* Aislar el dominio de los detalles técnicos
* Permitir testabilidad y evolución sin acoplamiento
* Facilitar el mantenimiento a largo plazo

---

## 🧩 Estructura típica (capas)

```
        +-----------------------------+
        |     Frameworks & Drivers    | ← Infra (Spring, DB, Kafka)
        +-----------------------------+
        |     Interface Adapters      | ← Controllers, Mappers, DTOs
        +-----------------------------+
        |      Application Layer      | ← UseCases, Ports
        +-----------------------------+
        |        Domain Layer         | ← Entities, VOs, Business Logic
        +-----------------------------+
```

---

## 🛠️ Componentes Clave

### 🔹 Domain Layer

* Núcleo de la aplicación: **Entities**, **Value Objects**, **Domain Services**
* **Sin dependencias externas**

```kotlin
data class Money(val amount: BigDecimal) {
  fun isPositive(): Boolean = amount >= BigDecimal.ZERO
}
```

---

### 🔹 Application Layer

* Define **casos de uso** del sistema
* Coordina entidades y puertos

```kotlin
class TransferMoneyUseCase(
  private val accountRepository: AccountRepository
) {
  fun execute(sourceId: UUID, targetId: UUID, amount: Money) {
    val source = accountRepository.find(sourceId)
    val target = accountRepository.find(targetId)
    source.withdraw(amount)
    target.deposit(amount)
    accountRepository.save(source)
    accountRepository.save(target)
  }
}
```

---

### 🔹 Interface Adapters

* Adaptan el mundo exterior a los casos de uso
* Aquí viven los **Controllers**, **Mappers**, **DTOs**, **Presenters**

```kotlin
@RestController
@RequestMapping("/accounts")
class AccountController(
  private val transferMoneyUseCase: TransferMoneyUseCase
) {
  @PostMapping("/transfer")
  fun transfer(@RequestBody req: TransferRequest) {
    transferMoneyUseCase.execute(req.from, req.to, Money(req.amount))
  }
}
```

---

### 🔹 Frameworks & Drivers

* Infraestructura técnica: Spring, JPA, Kafka, etc.
* Implementan puertos definidos en capas superiores

```kotlin
@Repository
class JpaAccountRepository(
  private val repo: SpringDataAccountRepository
) : AccountRepository {
  override fun find(id: UUID): Account = repo.findById(id).orElseThrow().toDomain()
  override fun save(account: Account) = repo.save(account.toEntity())
}
```

---

## 🔄 Relación entre capas

* **Dependencias solo apuntan hacia adentro**
* Infraestructura conoce casos de uso, pero los casos de uso no conocen la infraestructura
* Lo mismo para los controllers: conocen los casos de uso, no al revés

---

## ✅ Qué se espera de ti como consultor

* Saber estructurar un backend en estas capas con claridad
* Detectar cuándo hay **fugas de responsabilidad** (lógica en el controller, dominio acoplado a frameworks…)
* Ser capaz de **guiar equipos** en esta separación con convenciones claras
* Saber hacer refactors desde arquitecturas anémicas hacia clean
* Proponer nombres claros y roles explícitos en el código

---

## 💬 Ventajas clave

* Alta mantenibilidad
* Testabilidad sin necesidad de mocks externos
* Independencia de frameworks o decisiones técnicas
* Escalabilidad del diseño

---

## 🧪 Ejercicio para practicar

1. Define un caso de uso `BookHotelRoomUseCase` en la capa de aplicación
2. El dominio incluye `Booking`, `HotelRoom`, y `Money`
3. Implementa adaptadores: controller REST, repositorio JPA
4. Añade test unitario del caso de uso (sin Spring)

---

## 📚 Recursos

* Libro: *Clean Architecture* – Robert C. Martin
* Talk: [The Clean Architecture – Uncle Bob (YouTube)](https://www.youtube.com/watch?v=o_TH-Y78tt4)
* Proyecto ejemplo: [Kotlin Clean Architecture Example](https://github.com/bufferapp/android-clean-architecture-boilerplate)

---
