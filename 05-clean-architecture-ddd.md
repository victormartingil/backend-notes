# Clean Architecture & DDD - Guía de Entrevista Senior Backend

## 1. Clean Architecture

### Principios Fundamentales
- **Independencia**: Capas externas dependen de internas, nunca al revés
- **Testabilidad**: Lógica de negocio testeable sin dependencias externas
- **Flexibilidad**: Cambios en frameworks/DB no afectan lógica de negocio
- **Separación de responsabilidades**: Cada capa tiene una responsabilidad específica

### Capas de Clean Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                         │
│  (Web, DB, External APIs, Files, etc.)                         │
├─────────────────────────────────────────────────────────────────┤
│                 Interface Adapters                              │
│  (Controllers, Presenters, Gateways, etc.)                     │
├─────────────────────────────────────────────────────────────────┤
│                    Use Cases                                    │
│  (Application Business Rules)                                  │
├─────────────────────────────────────────────────────────────────┤
│                    Entities                                     │
│  (Enterprise Business Rules)                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Implementación en Spring Boot
```java
// Domain Layer - Entities
public class User {
    private final UserId id;
    private final Email email;
    private final Name name;
    private final Age age;
    private UserStatus status;
    
    public User(UserId id, Email email, Name name, Age age) {
        this.id = id;
        this.email = email;
        this.name = name;
        this.age = age;
        this.status = UserStatus.ACTIVE;
        
        // Domain rules
        if (age.getValue() < 18) {
            throw new InvalidUserAgeException("User must be at least 18 years old");
        }
    }
    
    public void deactivate() {
        if (this.status == UserStatus.DEACTIVATED) {
            throw new UserAlreadyDeactivatedException("User is already deactivated");
        }
        this.status = UserStatus.DEACTIVATED;
    }
}

// Value Objects
public class Email {
    private final String value;
    
    public Email(String value) {
        if (value == null || !value.contains("@")) {
            throw new InvalidEmailException("Invalid email format");
        }
        this.value = value;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Email)) return false;
        Email email = (Email) obj;
        return Objects.equals(value, email.value);
    }
}

// Repository Interface (Domain)
public interface UserRepository {
    Optional<User> findById(UserId id);
    Optional<User> findByEmail(Email email);
    User save(User user);
    void deleteById(UserId id);
}
```

## 2. Domain-Driven Design (DDD)

### Conceptos Clave
- **Bounded Context**: Límites donde un modelo es válido
- **Aggregates**: Conjunto de entidades tratadas como unidad
- **Aggregate Root**: Punto de entrada al agregado
- **Domain Services**: Lógica que no pertenece a ninguna entidad
- **Value Objects**: Objetos inmutables identificados por sus valores

### Implementación de Agregados
```java
// Aggregate Root
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final LocalDateTime createdAt;
    
    public Order(OrderId id, CustomerId customerId) {
        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>();
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }
    
    // Business method
    public void addItem(ProductId productId, Quantity quantity, Money price) {
        if (status != OrderStatus.PENDING) {
            throw new OrderCannotBeModifiedException("Order is not in pending status");
        }
        
        OrderItem item = new OrderItem(productId, quantity, price);
        items.add(item);
    }
    
    public void confirm() {
        if (items.isEmpty()) {
            throw new EmptyOrderException("Cannot confirm empty order");
        }
        
        if (status != OrderStatus.PENDING) {
            throw new OrderAlreadyConfirmedException("Order is already confirmed");
        }
        
        this.status = OrderStatus.CONFIRMED;
        
        // Domain event
        DomainEventPublisher.publish(new OrderConfirmedEvent(this.id, this.customerId));
    }
    
    public Money getTotalAmount() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Entity dentro del agregado
public class OrderItem {
    private final ProductId productId;
    private final Quantity quantity;
    private final Money unitPrice;
    
    public Money getSubtotal() {
        return unitPrice.multiply(quantity.getValue());
    }
}
```

### Domain Services
```java
@Component
public class OrderPricingService {
    
    public Money calculateOrderTotal(Order order, CustomerId customerId) {
        Money subtotal = order.getTotalAmount();
        
        // Apply customer discounts
        Optional<Discount> customerDiscount = discountRepository.findByCustomerId(customerId);
        if (customerDiscount.isPresent()) {
            subtotal = customerDiscount.get().apply(subtotal);
        }
        
        // Apply volume discounts
        if (order.getItems().size() > 10) {
            subtotal = subtotal.multiply(0.95); // 5% discount
        }
        
        return subtotal;
    }
}
```

## 3. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre Clean Architecture y Hexagonal Architecture?
- **Clean Architecture**: Enfoque en capas concéntricas con dependencias hacia adentro
- **Hexagonal Architecture**: Enfoque en puertos y adaptadores, núcleo aislado
- **Similitudes**: Ambas buscan independencia del framework y testabilidad
- **Diferencias**: Clean Architecture es más prescriptiva en sus capas

### ¿Qué es un Aggregate en DDD?
- **Definición**: Grupo de entidades tratadas como unidad de consistencia
- **Aggregate Root**: Única forma de acceder al agregado
- **Invariantes**: Reglas de negocio que deben mantenerse
- **Transacciones**: Una transacción = un agregado

### ¿Cuándo usar CQRS?
- **Casos de uso**: Sistemas con muchas consultas complejas
- **Escalabilidad**: Separar lectura y escritura
- **Complejidad**: Añade complejidad, usar solo cuando sea necesario
- **Event Sourcing**: Funciona bien con CQRS

### ¿Qué son los Value Objects?
- **Definición**: Objetos inmutables identificados por sus valores
- **Características**: Sin identidad, inmutables, intercambiables
- **Ejemplos**: Money, Email, Address, DateRange
- **Beneficios**: Tipo safety, encapsulación de lógica

## 4. Mejores Prácticas

### Domain Modeling
```java
// Hacer explícito el lenguaje del dominio
public class Account {
    public Money withdraw(Money amount) {
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException(
                "Cannot withdraw " + amount + " from account with balance " + balance
            );
        }
        
        this.balance = balance.subtract(amount);
        return amount;
    }
}

// Usar Value Objects para primitivos
public class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public boolean isLessThan(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) < 0;
    }
}
```

### Error Handling
```java
// Domain exceptions
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// Use case error handling
public class CreateUserUseCase {
    public User execute(CreateUserCommand command) {
        try {
            return userRepository.save(user);
        } catch (DataIntegrityViolationException e) {
            throw new EmailAlreadyExistsException("Email already exists");
        }
    }
}
```

## Puntos Clave para Recordar

1. **Dependency Rule**: Dependencias apuntan hacia el dominio
2. **Domain First**: Modelar el dominio antes que la infraestructura
3. **Immutability**: Preferir objetos inmutables especialmente Value Objects
4. **Single Responsibility**: Cada clase/método tiene una responsabilidad
5. **Testability**: Diseño testeable desde el principio
6. **Explicit**: Hacer explícito el lenguaje del dominio
7. **Boundaries**: Definir límites claros entre contextos
8. **Events**: Usar eventos para comunicación entre agregados
9. **Aggregates**: Mantener consistencia dentro de límites claros
10. **Ports & Adapters**: Aislar el core de detalles de implementación
