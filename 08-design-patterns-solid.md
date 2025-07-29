# Design Patterns & SOLID - Guía de Entrevista Senior Backend

## 1. Principios SOLID

### Single Responsibility Principle (SRP)
**Una clase debe tener una sola razón para cambiar**

```java
// ❌ Violación del SRP
public class User {
    private String name;
    private String email;
    
    public void setEmail(String email) { this.email = email; }
    public boolean isValidEmail() { return email.contains("@"); }
    public void saveToDatabase() { /* código BD */ }
}

// ✅ Respetando SRP
public class User {
    private String name;
    private String email;
    public void setEmail(String email) { this.email = email; }
}

public class EmailValidator {
    public boolean isValid(String email) {
        return email != null && email.contains("@");
    }
}

public class UserRepository {
    public void save(User user) { /* persistencia */ }
}
```

### Open/Closed Principle (OCP)
**Abierto para extensión, cerrado para modificación**

```java
// ❌ Violación del OCP
public class DiscountCalculator {
    public double calculateDiscount(String customerType, double amount) {
        if (customerType.equals("REGULAR")) return amount * 0.05;
        else if (customerType.equals("PREMIUM")) return amount * 0.10;
        return 0;
    }
}

// ✅ Respetando OCP
public interface DiscountStrategy {
    double calculateDiscount(double amount);
}

public class RegularCustomerDiscount implements DiscountStrategy {
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.05;
    }
}

public class DiscountCalculator {
    public double calculateDiscount(DiscountStrategy strategy, double amount) {
        return strategy.calculateDiscount(amount);
    }
}
```

### Dependency Inversion Principle (DIP)
**Depender de abstracciones, no de concreciones**

```java
// ❌ Violación del DIP
public class UserService {
    private EmailService emailService = new EmailService(); // Dependencia concreta
}

// ✅ Respetando DIP
public interface NotificationService {
    void sendNotification(String message);
}

public class UserService {
    private final NotificationService notificationService; // Dependencia abstracta
    
    public UserService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

## 2. Patrones Creacionales

### Singleton
```java
// Thread-safe Singleton
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    
    private DatabaseConnection() {}
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}

// Enum Singleton (recomendado)
public enum DatabaseConnection {
    INSTANCE;
    
    public void connect() { /* implementación */ }
}
```

### Builder
```java
public class User {
    private final String name;
    private final String email;
    private final int age;
    
    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
    }
    
    public static class Builder {
        private String name;
        private String email;
        private int age;
        
        public Builder setName(String name) {
            this.name = name;
            return this;
        }
        
        public Builder setEmail(String email) {
            this.email = email;
            return this;
        }
        
        public Builder setAge(int age) {
            this.age = age;
            return this;
        }
        
        public User build() {
            if (name == null || email == null) {
                throw new IllegalStateException("Name and email required");
            }
            return new User(this);
        }
    }
}

// Uso
User user = new User.Builder()
    .setName("John Doe")
    .setEmail("john@example.com")
    .setAge(30)
    .build();
```

## 3. Patrones Estructurales

### Adapter
```java
// Sistema externo incompatible
public class ExternalPaymentService {
    public void makePayment(String accountNumber, double amount) {
        System.out.println("External payment: $" + amount);
    }
}

// Nuestra interfaz
public interface PaymentService {
    void processPayment(String userId, double amount);
}

// Adapter
public class PaymentServiceAdapter implements PaymentService {
    private ExternalPaymentService externalService;
    
    @Override
    public void processPayment(String userId, double amount) {
        String accountNumber = "ACC_" + userId;
        externalService.makePayment(accountNumber, amount);
    }
}
```

### Decorator
```java
public interface Coffee {
    String getDescription();
    double getCost();
}

public class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() { return "Simple coffee"; }
    
    @Override
    public double getCost() { return 2.0; }
}

public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", milk";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.5;
    }
}

// Uso
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
```

## 4. Patrones Comportamentales

### Strategy
```java
public interface SortingStrategy {
    void sort(int[] array);
}

public class BubbleSort implements SortingStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Bubble sort");
    }
}

public class QuickSort implements SortingStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Quick sort");
    }
}

public class SortContext {
    private SortingStrategy strategy;
    
    public void setStrategy(SortingStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void executeStrategy(int[] array) {
        strategy.sort(array);
    }
}
```

### Observer
```java
public interface Observer {
    void update(String message);
}

public class NewsAgency {
    private List<Observer> observers = new ArrayList<>();
    
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
    
    public void setNews(String news) {
        notifyObservers(news);
    }
}

public class NewsChannel implements Observer {
    private String name;
    
    @Override
    public void update(String message) {
        System.out.println(name + " received: " + message);
    }
}
```

### Command
```java
public interface Command {
    void execute();
    void undo();
}

public class Light {
    private boolean on = false;
    
    public void turnOn() {
        on = true;
        System.out.println("Light ON");
    }
    
    public void turnOff() {
        on = false;
        System.out.println("Light OFF");
    }
}

public class TurnOnCommand implements Command {
    private Light light;
    
    @Override
    public void execute() { light.turnOn(); }
    
    @Override
    public void undo() { light.turnOff(); }
}

public class RemoteControl {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
    }
}
```

## 5. Preguntas Típicas de Entrevista

### ¿Diferencia entre Strategy y State?
- **Strategy**: Algoritmos intercambiables, cliente elige
- **State**: Comportamiento cambia según estado interno

### ¿Cuándo usar Singleton?
- **Casos válidos**: Loggers, cache, configuración
- **Problemas**: Testing difícil, acoplamiento global
- **Alternativas**: Dependency injection

### ¿Decorator vs Proxy?
- **Decorator**: Añade funcionalidad
- **Proxy**: Controla acceso

## 6. Mejores Prácticas

### Composition over Inheritance
```java
// ❌ Herencia rígida
public class Car extends Vehicle { }

// ✅ Composición flexible  
public class Car {
    private Engine engine;
    private Transmission transmission;
}
```

### Immutability
```java
public final class Money {
    private final BigDecimal amount;
    
    public Money add(Money other) {
        return new Money(this.amount.add(other.amount));
    }
}
```

## Puntos Clave para Recordar

1. **SOLID**: Principios fundamentales para código mantenible
2. **SRP**: Una clase, una responsabilidad
3. **OCP**: Extensible sin modificar código existente
4. **DIP**: Depender de abstracciones, no concreciones
5. **Patterns**: Soluciones probadas a problemas comunes
6. **Composition**: Preferir composición sobre herencia
7. **Immutability**: Objetos inmutables cuando sea posible
8. **Interfaces**: Programar contra interfaces
