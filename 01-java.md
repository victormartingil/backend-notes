# Java - Guía Completa Senior Backend

## 1. Tipos de Clases y Modificadores

### Clases Básicas
```java
// Clase final - no heredable
public final class Configuration {
    private final String value;
    
    public Configuration(String value) {
        this.value = value;
    }
}

// Clase abstracta - no instanciable
public abstract class Animal {
    protected String name;
    
    public Animal(String name) {
        this.name = name;
    }
    
    // Método abstracto
    public abstract String makeSound();
    
    // Método concreto
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Clase concreta
public class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }
    
    @Override
    public String makeSound() {
        return "Woof!";
    }
}
```

### Enums
```java
public enum UserRole {
    ADMIN("ADM", 3),
    EDITOR("EDT", 2),
    VIEWER("VWR", 1);
    
    private final String code;
    private final int level;
    
    UserRole(String code, int level) {
        this.code = code;
        this.level = level;
    }
    
    public boolean canEdit() {
        return level >= 2;
    }
    
    public static UserRole fromCode(String code) {
        return Arrays.stream(values())
            .filter(role -> role.code.equals(code))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Invalid code: " + code));
    }
}
```

### Records (Java 14+)
```java
// Record básico para DTOs
public record UserDto(
    Long id,
    String name,
    String email,
    boolean active
) {
    // Constructor compacto para validaciones
    public UserDto {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
    
    // Métodos adicionales
    public boolean isValid() {
        return name != null && email != null && email.contains("@");
    }
}

// Record para Value Objects
public record UserId(Long value) {
    public UserId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("UserId must be positive");
        }
    }
}
```

## 2. Programación Funcional

### Interfaces Funcionales
```java
public class FunctionalInterfacesExample {
    public void predicate() {
        // Predicate<T> - toma T, retorna boolean
        Predicate<String> isEmpty = String::isEmpty;
        Predicate<String> isNotEmpty = isEmpty.negate();
        Predicate<String> startsWith = s -> s.startsWith("A");
        
        // Combinar predicados
        Predicate<String> combined = isEmpty.or(startsWith);
        
        List<String> names = Arrays.asList("Alice", "", "Bob", "Anna");
        List<String> filtered = names.stream()
            .filter(isNotEmpty.and(startsWith))
            .collect(Collectors.toList());
    }
    
    public void function() {
        // Function<T, R> - toma T, retorna R
        Function<String, Integer> length = String::length;
        Function<Integer, String> toString = Object::toString;
        
        // Componer funciones
        Function<String, String> lengthToString = length.andThen(toString);
    }
    
    public void consumer() {
        // Consumer<T> - toma T, no retorna nada
        Consumer<String> print = System.out::println;
        Consumer<String> upperPrint = s -> System.out.println(s.toUpperCase());
        
        // Combinar consumers
        Consumer<String> combined = print.andThen(upperPrint);
    }
    
    public void supplier() {
        // Supplier<T> - no toma nada, retorna T
        Supplier<String> randomString = () -> UUID.randomUUID().toString();
        Supplier<LocalDateTime> now = LocalDateTime::now;
        
        // Lazy evaluation
        Optional<String> value = Optional.empty();
        String result = value.orElseGet(randomString);
    }
}
```

### Streams API Avanzado
```java
public class StreamsAdvanced {
    public void advancedCollectors() {
        List<Person> people = Arrays.asList(
            new Person("Alice", 30, "alice@example.com"),
            new Person("Bob", 25, "bob@example.com"),
            new Person("Charlie", 35, "charlie@example.com")
        );
        
        // Agrupación
        Map<String, List<Person>> byName = people.stream()
            .collect(Collectors.groupingBy(Person::getName));
        
        // Agrupación con downstream collector
        Map<String, Long> countByName = people.stream()
            .collect(Collectors.groupingBy(Person::getName, Collectors.counting()));
        
        // Partición
        Map<Boolean, List<Person>> adultPartition = people.stream()
            .collect(Collectors.partitioningBy(p -> p.getAge() >= 30));
        
        // Joining
        String allNames = people.stream()
            .map(Person::getName)
            .collect(Collectors.joining(", ", "[", "]"));
        
        // Collectors.toMap
        Map<String, Integer> nameToAge = people.stream()
            .collect(Collectors.toMap(
                Person::getName,
                Person::getAge,
                (existing, replacement) -> existing
            ));
        
        // Estadísticas
        IntSummaryStatistics ageStats = people.stream()
            .mapToInt(Person::getAge)
            .summaryStatistics();
    }
}
```

## 3. Optional

### Uso Correcto de Optional
```java
public class OptionalExamples {
    public void basicUsage() {
        // Crear Optional
        Optional<String> empty = Optional.empty();
        Optional<String> nonEmpty = Optional.of("value");
        Optional<String> nullable = Optional.ofNullable(getString());
        
        // Mejor: usar ifPresent
        nonEmpty.ifPresent(System.out::println);
        
        // Valor por defecto
        String value = empty.orElse("default");
        String lazyValue = empty.orElseGet(() -> computeDefault());
        
        // Lanzar excepción si vacío
        String required = empty.orElseThrow(() -> 
            new IllegalStateException("Value is required"));
    }
    
    public void transformations() {
        Optional<String> optional = Optional.of("hello");
        
        // map - transformar valor si presente
        Optional<Integer> length = optional.map(String::length);
        Optional<String> upper = optional.map(String::toUpperCase);
        
        // flatMap - para Optional anidados
        Optional<String> result = optional.flatMap(this::processString);
        
        // filter - filtrar valor
        Optional<String> filtered = optional.filter(s -> s.startsWith("h"));
    }
    
    public void chainingOperations() {
        Optional<Person> person = findPerson("Alice");
        
        // Encadenar operaciones
        String email = person
            .filter(p -> p.getAge() > 18)
            .map(Person::getEmail)
            .filter(e -> e.contains("@"))
            .map(String::toLowerCase)
            .orElse("no-email@example.com");
    }
    
    private String getString() {
        return Math.random() > 0.5 ? "value" : null;
    }
    
    private String computeDefault() {
        return "computed_default";
    }
    
    private Optional<String> processString(String s) {
        return s.length() > 3 ? Optional.of(s.toUpperCase()) : Optional.empty();
    }
    
    private Optional<Person> findPerson(String name) {
        return name.equals("Alice") ? 
            Optional.of(new Person("Alice", 30, "alice@example.com")) : 
            Optional.empty();
    }
}
```

## 4. Manejo de Excepciones

### Try-Catch-Finally y Try-with-Resources
```java
public class ExceptionHandling {
    public void basicTryCatch() {
        try {
            riskyOperation();
        } catch (SpecificException e) {
            logger.error("Specific error occurred", e);
        } catch (RuntimeException e) {
            logger.error("Runtime error", e);
        } catch (Exception e) {
            logger.error("General error", e);
        } finally {
            cleanup();
        }
    }
    
    public void multiCatch() {
        try {
            complexOperation();
        } catch (IOException | SQLException e) {
            logger.error("IO or SQL error", e);
        } catch (Exception e) {
            logger.error("Other error", e);
        }
    }
    
    public String tryWithResources() {
        try (FileInputStream fis = new FileInputStream("file.txt");
             BufferedReader reader = new BufferedReader(new InputStreamReader(fis))) {
            
            return reader.readLine();
            
        } catch (IOException e) {
            logger.error("Error reading file", e);
            throw new ServiceException("Failed to read file", e);
        }
    }
}

// Excepciones personalizadas
class SpecificException extends Exception {
    public SpecificException(String message) {
        super(message);
    }
    
    public SpecificException(String message, Throwable cause) {
        super(message, cause);
    }
}

class ServiceException extends RuntimeException {
    public ServiceException(String message) {
        super(message);
    }
    
    public ServiceException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Result Pattern para Evitar Excepciones
```java
// Result type para manejo funcional de errores
public abstract class Result<T> {
    public abstract boolean isSuccess();
    public abstract boolean isFailure();
    public abstract T getValue();
    public abstract Throwable getError();
    
    public static <T> Result<T> success(T value) {
        return new Success<>(value);
    }
    
    public static <T> Result<T> failure(Throwable error) {
        return new Failure<>(error);
    }
    
    public static <T> Result<T> failure(String message) {
        return new Failure<>(new RuntimeException(message));
    }
    
    // Map para transformar valor en caso de éxito
    public abstract <R> Result<R> map(Function<T, R> mapper);
    
    // FlatMap para operaciones que retornan Result
    public abstract <R> Result<R> flatMap(Function<T, Result<R>> mapper);
    
    // Fold para manejar ambos casos
    public abstract <R> R fold(Function<T, R> onSuccess, Function<Throwable, R> onFailure);
    
    // OnSuccess/OnFailure para side effects
    public abstract Result<T> onSuccess(Consumer<T> action);
    public abstract Result<T> onFailure(Consumer<Throwable> action);
    
    // GetOrElse para valor por defecto
    public abstract T getOrElse(T defaultValue);
    public abstract T getOrElse(Supplier<T> defaultSupplier);
}
```

## 5. Spring Boot Integration

### Controllers REST
```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping
    public ResponseEntity<PageResponse<UserDto>> getUsers(
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "10") @Min(1) @Max(100) int size,
            @RequestParam(required = false) String name,
            @RequestParam(required = false) Boolean active) {
        
        log.info("Getting users - page: {}, size: {}, name: {}, active: {}", page, size, name, active);
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        Page<UserDto> users = userService.findUsers(name, active, pageable);
        
        PageResponse<UserDto> response = PageResponse.of(users);
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        log.info("Getting user with id: {}", id);
        
        return userService.findById(id)
            .map(user -> ResponseEntity.ok(user))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<UserDto> createUser(
            @Valid @RequestBody CreateUserRequest request,
            HttpServletRequest httpRequest) {
        
        log.info("Creating user: {}", request);
        
        UserDto createdUser = userService.createUser(request);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(createdUser.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(createdUser);
    }
}
```

### DTOs con Validación
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CreateUserRequest {
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email format is invalid")
    @Size(max = 100, message = "Email cannot exceed 100 characters")
    private String email;
    
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must be at most 120")
    private Integer age;
    
    @Pattern(regexp = "^[A-Z]{2}$", message = "Country code must be 2 uppercase letters")
    private String countryCode;
    
    @Valid
    @NotNull(message = "Address is required")
    private AddressDto address;
    
    @Size(max = 3, message = "Maximum 3 hobbies allowed")
    private List<@NotBlank(message = "Hobby cannot be blank") String> hobbies;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class AddressDto {
    
    @NotBlank(message = "Street is required")
    private String street;
    
    @NotBlank(message = "City is required")
    private String city;
    
    @Pattern(regexp = "^\\d{5}$", message = "ZIP code must be 5 digits")
    private String zipCode;
}
```

## 6. Testing Profesional

### Testing con JUnit 5
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private UserMapper userMapper;
    
    @Mock
    private ApplicationEventPublisher eventPublisher;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    @DisplayName("Should create user when valid request is provided")
    void shouldCreateUserWhenValidRequest() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("John Doe")
            .email("john@example.com")
            .age(30)
            .build();
        
        User savedUser = User.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .age(30)
            .active(true)
            .createdAt(LocalDateTime.now())
            .build();
        
        UserDto expectedDto = UserDto.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .age(30)
            .active(true)
            .build();
        
        when(userRepository.existsByEmail(request.getEmail())).thenReturn(false);
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        when(userMapper.toDto(savedUser)).thenReturn(expectedDto);
        
        // When
        UserDto result = userService.createUser(request);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("John Doe");
        assertThat(result.getEmail()).isEqualTo("john@example.com");
        assertThat(result.isActive()).isTrue();
        
        verify(userRepository).existsByEmail("john@example.com");
        verify(userRepository).save(any(User.class));
        verify(eventPublisher).publishEvent(any(UserCreatedEvent.class));
    }
    
    @Test
    @DisplayName("Should throw DuplicateEmailException when email already exists")
    void shouldThrowExceptionWhenEmailExists() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("John Doe")
            .email("existing@example.com")
            .age(30)
            .build();
        
        when(userRepository.existsByEmail(request.getEmail())).thenReturn(true);
        
        // When & Then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessageContaining("Email already exists");
        
        verify(userRepository).existsByEmail("existing@example.com");
        verify(userRepository, never()).save(any(User.class));
        verify(eventPublisher, never()).publishEvent(any());
    }
}
```

## 7. Resumen de Conceptos Clave

### Tipos de Clases
- **`class`**: Clase concreta instanciable
- **`abstract class`**: No instanciable, puede tener implementación parcial
- **`final class`**: No heredable
- **`enum`**: Conjunto fijo de constantes con comportamiento
- **`record`**: Clases inmutables para datos (Java 14+)
- **Clases anidadas**: `static`, `inner`, `local`, `anonymous`

### Modificadores de Visibilidad
- **`public`**: Accesible desde cualquier lugar
- **`protected`**: Accesible en paquete y subclases
- **`package-private`**: Accesible solo en el paquete
- **`private`**: Accesible solo en la clase

### Variables y Tipos
- **Primitivos**: `int`, `boolean`, `double`, etc.
- **Referencias**: Objetos, arrays
- **`final`**: No reasignable
- **`static`**: Pertenece a la clase, no a la instancia
- **`var`**: Inferencia de tipos (Java 10+)

### Interfaces Funcionales
- **`Predicate<T>`**: T → boolean
- **`Function<T,R>`**: T → R
- **`Consumer<T>`**: T → void
- **`Supplier<T>`**: () → T
- **`BiFunction<T,U,R>`**: (T,U) → R

### Streams API
- **Operaciones intermedias**: `filter`, `map`, `sorted` (lazy)
- **Operaciones terminales**: `collect`, `forEach`, `reduce` (eager)
- **Collectors**: `toList`, `groupingBy`, `joining`
- **Parallel streams**: Para procesamiento paralelo

### Optional
- **Creación**: `of`, `ofNullable`, `empty`
- **Verificación**: `isPresent`, `isEmpty`
- **Transformación**: `map`, `flatMap`, `filter`
- **Valores por defecto**: `orElse`, `orElseGet`, `orElseThrow`
- **Side effects**: `ifPresent`, `ifPresentOrElse`

### Manejo de Excepciones
- **Checked**: Deben ser manejadas o declaradas (`IOException`, `SQLException`)
- **Unchecked**: Runtime exceptions (`RuntimeException`, `NullPointerException`)
- **Try-with-resources**: Cierre automático de recursos
- **Multi-catch**: Manejar múltiples excepciones en un bloque
- **Custom exceptions**: Crear excepciones específicas del dominio

### Spring Boot
- **Stereotypes**: `@Service`, `@Repository`, `@Controller`, `@Component`
- **DI**: Constructor injection (recomendado), field injection
- **REST**: `@RestController`, `@GetMapping`, `@PostMapping`
- **Validation**: `@Valid`, `@NotNull`, `@Email`, `@Size`
- **Exception handling**: `@ControllerAdvice`, `@ExceptionHandler`
- **Events**: `@EventListener`, `ApplicationEventPublisher`

### Testing
- **JUnit 5**: `@Test`, `@ParameterizedTest`, `@Nested`
- **Mockito**: `@Mock`, `@InjectMocks`, `when().thenReturn()`
- **AssertJ**: Fluent assertions más expresivas
- **Spring Test**: `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`
- **Testcontainers**: Testing con bases de datos reales

### Mejores Prácticas
- **Inmutabilidad**: Usar `final`, records, builder pattern
- **Null safety**: Usar `Optional`, validaciones defensivas
- **Exception handling**: Específico antes que genérico
- **Streams**: Operaciones declarativas sobre colecciones
- **Dependency injection**: Constructor injection
- **Single Responsibility**: Una clase, una responsabilidad
- **Clean Code**: Nombres descriptivos, métodos pequeños

## Puntos Clave para Recordar

1. **OOP**: Herencia, polimorfismo, encapsulación, abstracción
2. **Functional Programming**: Lambdas, streams, Optional
3. **Memory Management**: Stack vs heap, garbage collection
4. **Collections**: Complejidad temporal, thread safety
5. **Concurrency**: CompletableFuture para asíncrono
6. **Spring Boot**: Inyección de dependencias, REST APIs
7. **Testing**: Unit tests, integration tests, mocking
8. **Error Handling**: Try-catch, custom exceptions, Result pattern
9. **Modern Java**: Records, var, pattern matching
10. **Clean Architecture**: Separación en capas, SOLID principles
