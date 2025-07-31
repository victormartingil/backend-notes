# Arquitectura Hexagonal + DDD

Este documento combina de forma concisa los principios esenciales de Clean Architecture, Hexagonal Architecture y Domain-Driven Design (DDD), con ejemplos prácticos en Java y Kotlin.

---

## 1. Fundamentos de Arquitectura

### 🧱 Clean Architecture

* **Independencia**: las capas externas dependen de las internas, nunca al revés.
* **Testabilidad**: la lógica de negocio no depende de frameworks.
* **Separación de responsabilidades** clara por capas.
* **Flexibilidad tecnológica**: puedes sustituir DB, framework o UI sin afectar el core.

```
┌─────────────────────────────────────────────────────┐
│              Frameworks & Drivers                   │ ← Web, DB, APIs externas
├─────────────────────────────────────────────────────┤
│              Interface Adapters                     │ ← Controllers, Presenters
├─────────────────────────────────────────────────────┤
│                  Use Cases                          │ ← Application services
├─────────────────────────────────────────────────────┤
│                  Entities                           │ ← Core domain (DDD)
└─────────────────────────────────────────────────────┘
```

### 🧩 Hexagonal Architecture (Ports & Adapters)

* **Puertos** (interfaces) definen qué necesita el dominio.
* **Adaptadores** (implementaciones) concretan cómo.
* Los adaptadores pueden ser de entrada (REST, Kafka...) o de salida (DB, Email, Cola).
* El dominio **no conoce ningún framework**.

```kotlin
// Puerto de salida (dominio)
interface EmailSender {
    fun send(to: EmailAddress, subject: String, body: String)
}

// Adaptador (infraestructura)
class EmailSenderImpl(private val eventStore: EventStoreRepository) : EmailSender {
    override fun send(to: EmailAddress, subject: String, body: String) {
        eventStore.store(EmailEvent(to.value, subject, body))
    }
}
```

> ✅ Incluso si hay solo una implementación, **se mantiene la interfaz** por principios de arquitectura: desacoplamiento, testabilidad y claridad.

---

## 2. Tipos de Objetos en DDD (con ejemplos Java/Kotlin)

### Entidad (Entity)
Objeto con identidad propia que perdura en el tiempo, incluso si cambian sus atributos.
Ejemplo: Un usuario registrado en tu sistema que puede cambiar su email pero mantiene el mismo identificador único.

Cuándo usarlo:
Cuando necesitas distinguir claramente diferentes objetos del mismo tipo mediante una identidad persistente (ej.: usuarios, productos, pedidos).

```java
public class User {
    private final UserId id;
    private Email email;
    private UserStatus status;

    public void deactivate() {
        this.status = UserStatus.DEACTIVATED;
    }
}
```

```kotlin
class User(val id: UUID, var email: EmailAddress, var status: UserStatus)
```

### Objeto de Valor (Value Object)
Objeto inmutable que se compara por sus atributos (no tiene identidad propia).
Ejemplo: El valor monetario de un pedido o una dirección física.

Cuándo usarlo:
Cuando modelas conceptos que se definen por sus atributos y no cambian, como emails, direcciones, importes monetarios, etc.

```java
public class Email {
    private final String value;
    public Email(String value) {
        if (!value.contains("@")) throw new InvalidEmailException();
        this.value = value;
    }
}
```

```kotlin
@JvmInline value class EmailAddress(val value: String)
```

### Agregado
Conjunto de entidades y objetos de valor que forman una unidad lógica y consistente, accesible solo desde una entidad raíz (Aggregate Root).
Ejemplo: Un pedido (Order) con sus líneas de pedido (OrderItems).

Cuándo usarlo:
Cuando tienes reglas de negocio complejas que implican múltiples objetos relacionados, asegurando su integridad como conjunto (pedidos con múltiples artículos, una factura con sus líneas, etc.).

```java
public class Order {
    private final OrderId id;
    private final List<OrderItem> items;

    public void confirm() {
        if (items.isEmpty()) throw new EmptyOrderException();
        // Confirma el pedido
    }
}
```

```kotlin
class Order(val id: UUID, private val items: List<OrderItem>) {
    fun confirm() {
        require(items.isNotEmpty()) { "Order cannot be empty" }
        // Confirmar pedido
    }
}
```

### Factory (Factoría)
Objeto especializado en construir entidades, objetos de valor o agregados complejos, garantizando reglas de negocio y consistencia.
Ejemplo: Una Factory que construye un pedido, validando que siempre tenga al menos un artículo.

Cuándo usarlo:  
Cuando la construcción del objeto tiene reglas de negocio complejas o validaciones que no deben repetirse en cada lugar que se crea dicho objeto.

```java
public class OrderFactory {
    public static Order create(List<OrderItem> items) {
        if(items.isEmpty()) throw new IllegalArgumentException("Order must have at least one item.");
        return new Order(OrderId.randomId(), items);
    }
}
```

```kotlin
object OrderFactory {
    fun create(items: List<OrderItem>): Order {
        require(items.isNotEmpty()) { "Order must have at least one item." }
        return Order(UUID.randomUUID(), items)
    }
}

```

### Evento de Dominio
Representa algo importante que ocurrió dentro del dominio, permitiendo que otras partes del sistema reaccionen al cambio.
Ejemplo: Cuando se crea un pedido, se lanza un evento para notificar otros sistemas (facturación, logística).

Cuándo usarlo:
Para comunicar cambios relevantes en tu dominio, facilitando un sistema desacoplado y reactivo (ej. confirmar la creación de pedidos, pagos exitosos).

```java
public class OrderCreatedEvent {
    private final OrderId orderId;

    public OrderCreatedEvent(OrderId orderId) {
        this.orderId = orderId;
    }
    // publicar al manejador de eventos
}
```

```kotlin
data class OrderCreated(val orderId: UUID) : DomainEvent
```

### Servicio de Dominio
Lógica que no encaja naturalmente en una única entidad o agregado, pero pertenece claramente al dominio.
Ejemplo: Servicio para calcular impuestos complejos o verificar contraseñas fuertes.

Cuándo usarlo:
Cuando una lógica involucra múltiples entidades o agregados y no tiene un dueño claro, como cálculos de precios o validaciones especiales.

```java
public class OrderPricingService {
    public Money calculateTotal(Order order) {
        // calcular impuestos y descuentos
    }
}
```

```kotlin
interface PasswordStrengthChecker {
    fun isStrong(password: String): Boolean
}
```

### Repositorio
Abstracción para almacenar y recuperar agregados del almacenamiento persistente (ej.: base de datos).
Ejemplo: Recuperar o guardar usuarios en base de datos.

Cuándo usarlo:
Siempre que necesites acceder a datos persistentes (bases de datos, etc.) de forma desacoplada del dominio.

```java
public interface UserRepository {
    Optional<User> findById(UserId id);
}
```

```kotlin
interface UserRepository {
    fun findById(id: UUID): User?
    fun save(user: User)
}
```

### Caso de Uso (Application Service)
Orquesta un flujo específico de negocio, utilizando entidades, servicios de dominio y repositorios para llevar a cabo una operación completa.
Ejemplo: Registrar un nuevo usuario y enviarle un email de bienvenida.

Cuándo usarlo:
Cuando necesitas organizar claramente los pasos de una operación de negocio, manteniendo la lógica separada del dominio.

```java
public class CreateUserUseCase {
    private final UserRepository repository;
    private final EmailSender emailSender;

    public void execute(CreateUserCommand command) {
        User user = new User(command.id(), command.email());
        repository.save(user);
        emailSender.sendWelcome(user);
    }
}
```

```kotlin
class RegisterUserUseCase(
    private val repo: UserRepository,
    private val emailSender: EmailSender
) {
    fun execute(request: RegisterUserRequestDto) {
        val user = User(UUID.randomUUID(), EmailAddress(request.email), UserStatus.ACTIVE)
        repo.save(user)
        emailSender.sendWelcome(user)
    }
}
```

### DTO (Data Transfer Object)
Objeto simple usado exclusivamente para transportar información entre capas (presentación, aplicación, infraestructura).
Ejemplo: Recibir datos de registro de usuario desde un formulario.

Cuándo usarlo:
Cuando quieres pasar información entre frontend-backend, o entre capas del backend, sin exponer directamente entidades o lógica de negocio.

```java
public class RegisterUserRequestDto {
    private final String email;
    private final String password;
    // getters y setters
}
```

```kotlin
data class RegisterUserRequestDto(val email: String, val password: String)
```

---

## 3. Estructura de Carpetas Sugerida (Hexagonal + DDD)

```
src/main/kotlin/com/example/project/
├── domain/
│   ├── user/
│   │   ├── User.kt
│   │   ├── Email.kt
│   │   └── event/
│   │       └── UserRegistered.kt
│   └── order/
│       ├── Order.kt
│       ├── OrderItem.kt
│       └── OrderFactory.kt
├── application/
│   └── user/
│       ├── RegisterUserUseCase.kt
│       └── dto/
│           └── RegisterUserRequestDto.kt
├── infrastructure/
│   ├── user/
│   │   └── JpaUserRepository.kt
│   └── notification/
│       └── EmailSenderImpl.kt
└── interface/
    ├── rest/
    │   └── UserController.kt
    └── event/
        └── KafkaEventConsumer.kt
```

---

## 4. Preguntas Clave para Entrevistas Técnicas

### 🔁 Clean vs Hexagonal Architecture

* Clean es más prescriptiva por capas.
* Hexagonal se centra en entradas/salidas bien definidas (puertos).

### 🧱 ¿Qué es un Aggregate?

* Unidad de consistencia con reglas de negocio internas.
* Solo puede ser accedido a través de su raíz.

### 🚀 ¿Cuándo usar CQRS?

* Muchas lecturas con proyecciones específicas.
* Escenarios donde separar lectura/escritura aporta escalabilidad.

### 🧾 ¿Qué son los Value Objects?

* Tipos inmutables como Email, Money, Address.
* Mejora claridad, seguridad de tipos y expresividad.

---

## 5. Buenas Prácticas para Backend Senior

* Modelar el dominio con nombres del lenguaje del negocio.

```kotlin
class BankAccount(val balance: Money) {
    fun withdraw(amount: Money): Money {
        require(balance >= amount) { "Insufficient funds" }
        return amount
    }
}
```

* Preferir Value Objects sobre tipos primitivos.

```java
// En lugar de usar String para Email
public class Email {
    private final String value;
    public Email(String value) {
        if (!value.contains("@")) throw new InvalidEmailException();
        this.value = value;
    }
}
```

* Usar puertos para externalizar infraestructura.

```kotlin
interface NotificationSender {
    fun send(message: String)
}
```

* Mantener el dominio libre de dependencias.

```kotlin
// No debe haber imports de Spring, JPA, etc. en esta clase
class Product(val id: UUID, val name: String)
```

* Lógica de negocio = dominio, no en controladores.

```kotlin
// ❌ Antipatrón: lógica en el controller
@PostMapping("/withdraw")
fun withdraw(...) {
    if (amount > balance) throw IllegalArgumentException("Insufficient funds")
}
```

```kotlin
// ✅ Correcto: lógica en el dominio
class Account(val balance: Money) {
    fun withdraw(amount: Money): Money {
        if (balance < amount) throw IllegalArgumentException("Insufficient funds")
        return amount
    }
}
```

* Testing unitario en dominio, integración en adaptadores.

```kotlin
@Test
fun `should withdraw money if balance sufficient`() {
    val account = BankAccount(balance = Money(100))
    val withdrawn = account.withdraw(Money(50))
    assertEquals(Money(50), withdrawn)
}
```

* Usar eventos para comunicar cambios dentro del dominio.

```kotlin
data class PaymentCompleted(val paymentId: UUID) : DomainEvent
```

* Aplicar principios **SOLID** para mantener un código mantenible y extensible.


* Usar un **Anticorruption Layer (ACL)** para aislar el dominio de integraciones o modelos externos.<br>
  Un Anticorruption Layer (ACL) es una capa de adaptación que protege tu dominio de modelos y contratos externos.
  No solo mapea datos: también puede validar, transformar, combinar información y aislar tu lógica de negocio de cambios externos.
  Es la frontera que garantiza que tu dominio solo trabaje con su propio lenguaje y estructuras, nunca con detalles ajenos.
```java
// DTO de sistema externo
public class ExternalOrderDto {
    public String code;
    public double amount;
    public String createdAt;
}

// Entidad de tu dominio
public class Order {
    private final OrderId id;
    private final Money total;
    private final LocalDate createdAt;

    public Order(OrderId id, Money total, LocalDate createdAt) {
        this.id = id;
        this.total = total;
        this.createdAt = createdAt;
    }
}

// ACL: traduce el modelo externo a tu dominio
public class ExternalOrderAcl {

    public Order toDomainOrder(ExternalOrderDto external) {
        return new Order(
            OrderId.fromExternalCode(external.code),
            new Money(external.amount),
            LocalDate.parse(external.createdAt)
        );
    }
}
```

```kotlin
// DTO de sistema externo
data class ExternalPaymentEvent(
    val payment_code: String,
    val paid_amount: Double,
    val date: String
)

// Evento de dominio propio
data class PaymentCompleted(
    val paymentId: PaymentId,
    val amount: Money,
    val date: LocalDate
)

// ACL que transforma datos externos a tu modelo de dominio
class ExternalPaymentAcl {
    fun toDomainEvent(event: ExternalPaymentEvent): PaymentCompleted {
        return PaymentCompleted(
            paymentId = PaymentId(event.payment_code),
            amount = Money(event.paid_amount),
            date = LocalDate.parse(event.date)
        )
    }
}
```

---
