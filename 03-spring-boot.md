# Spring Boot - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### ¿Qué es Spring Boot?
- **Framework**: Construido sobre Spring Framework
- **Opinionated**: Configuración automática con defaults sensatos
- **Standalone**: Aplicaciones independientes con servidor embebido
- **Production-ready**: Métricas, health checks, configuración externa

### Ventajas de Spring Boot
- **Auto-configuración**: Reduce configuración manual
- **Starter dependencies**: Conjuntos de dependencias preconfiguradas
- **Embedded servers**: Tomcat, Jetty, Undertow
- **Actuator**: Monitoreo y métricas
- **Profiles**: Configuración por entorno

## 2. Configuración y Profiles

### application.properties vs application.yml
```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.hibernate.ddl-auto=create-drop
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    hibernate:
      ddl-auto: create-drop
```

### Profiles
```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
  logging:
    level:
      com.example: DEBUG

# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/myapp
  logging:
    level:
      root: WARN
```

```java
// Activar profiles
@ActiveProfiles("dev")
@Component
public class DevConfiguration {
    // Solo activo en profile 'dev'
}

@Profile("prod")
@Configuration
public class ProductionConfig {
    // Configuración específica de producción
}
```

## 3. Dependency Injection y IoC

### Anotaciones de Inyección
```java
@RestController
public class UserController {
    
    // Constructor injection (recomendado)
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    // Field injection (no recomendado)
    @Autowired
    private UserRepository userRepository;
    
    // Setter injection
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

### Configuración con @Configuration
```java
@Configuration
public class AppConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://localhost/primary")
            .build();
    }
    
    @Bean
    @Qualifier("secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://localhost/secondary")
            .build();
    }
}
```

### Scopes de Beans
```java
@Component
@Scope("singleton") // Por defecto
public class SingletonService { }

@Component
@Scope("prototype") // Nueva instancia cada vez
public class PrototypeService { }

@Component
@Scope("request") // Web scope
public class RequestScopedService { }
```

## 4. Spring MVC

### Controladores REST
```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {
    
    @GetMapping
    public ResponseEntity<List<UserDTO>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        Pageable pageable = PageRequest.of(page, size);
        Page<User> users = userService.findAll(pageable);
        
        return ResponseEntity.ok(
            users.map(userMapper::toDTO).getContent()
        );
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(userMapper::toDTO)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<UserDTO> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        
        User user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userMapper.toDTO(user));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        
        return userService.update(id, request)
            .map(userMapper::toDTO)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Validación
```java
public class CreateUserRequest {
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
    
    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must not exceed 120")
    private Integer age;
    
    // Getters y setters
}
```

### Exception Handling
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex) {
        
        ErrorResponse error = new ErrorResponse(
            "USER_NOT_FOUND",
            ex.getMessage(),
            LocalDateTime.now()
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());
        
        ErrorResponse error = new ErrorResponse(
            "VALIDATION_ERROR",
            "Validation failed",
            errors,
            LocalDateTime.now()
        );
        
        return ResponseEntity.badRequest().body(error);
    }
}
```

## 5. Spring Data JPA

### Repositorios
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Query methods
    List<User> findByName(String name);
    List<User> findByAgeGreaterThan(int age);
    Optional<User> findByEmail(String email);
    List<User> findByNameContainingIgnoreCase(String name);
    
    // Custom queries
    @Query("SELECT u FROM User u WHERE u.age BETWEEN :minAge AND :maxAge")
    List<User> findByAgeBetween(@Param("minAge") int minAge, @Param("maxAge") int maxAge);
    
    @Query(value = "SELECT * FROM users WHERE created_at > :date", nativeQuery = true)
    List<User> findRecentUsers(@Param("date") LocalDateTime date);
    
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoffDate")
    int deactivateInactiveUsers(@Param("cutoffDate") LocalDateTime cutoffDate);
    
    // Projections
    List<UserProjection> findByActive(boolean active);
    
    @Query("SELECT new com.example.dto.UserSummary(u.id, u.name, u.email) FROM User u")
    List<UserSummary> findUserSummaries();
}

// Projection interface
interface UserProjection {
    Long getId();
    String getName();
    String getEmail();
}
```

### Entidades JPA
```java
@Entity
@Table(name = "users")
@EntityListeners(AuditingEntityListener.class)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 50)
    private String name;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private Integer age;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @Version
    private Long version; // Optimistic locking
    
    // Relaciones
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
    
    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    // Constructores, getters, setters
}
```

### Transacciones
```java
@Service
@Transactional(readOnly = true)
public class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    @Transactional
    public User createUser(CreateUserRequest request) {
        // Validar que el email no existe
        if (userRepository.findByEmail(request.getEmail()).isPresent()) {
            throw new EmailAlreadyExistsException("Email already exists");
        }
        
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user.setAge(request.getAge());
        user.setStatus(UserStatus.ACTIVE);
        
        User savedUser = userRepository.save(user);
        
        // Enviar email de bienvenida
        try {
            emailService.sendWelcomeEmail(savedUser.getEmail());
        } catch (Exception e) {
            log.error("Failed to send welcome email", e);
        }
        
        return savedUser;
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logUserActivity(Long userId, String activity) {
        // Esta transacción es independiente
        UserActivity log = new UserActivity(userId, activity, LocalDateTime.now());
        userActivityRepository.save(log);
    }
}
```

## 6. Spring Security

### Configuración Básica
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtRequestFilter jwtRequestFilter;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/users").hasRole("USER")
                .requestMatchers(HttpMethod.POST, "/api/users").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

### JWT Implementation
```java
@Component
public class JwtUtil {
    
    private String secret = "mySecretKey";
    private int jwtExpiration = 86400; // 24 hours
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(subject)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration * 1000))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
    
    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
    }
    
    private Boolean isTokenExpired(String token) {
        final Date expiration = getExpirationDateFromToken(token);
        return expiration.before(new Date());
    }
}
```

## 7. Testing

### Unit Tests
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void shouldCreateUserSuccessfully() {
        // Given
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com", 25);
        User savedUser = new User(1L, "John", "john@example.com", 25);
        
        when(userRepository.findByEmail(request.getEmail()))
            .thenReturn(Optional.empty());
        when(userRepository.save(any(User.class)))
            .thenReturn(savedUser);
        
        // When
        User result = userService.createUser(request);
        
        // Then
        assertThat(result.getName()).isEqualTo("John");
        assertThat(result.getEmail()).isEqualTo("john@example.com");
        
        verify(userRepository).findByEmail(request.getEmail());
        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("john@example.com");
    }
}
```

### Integration Tests
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(locations = "classpath:application-test.properties")
@Transactional
class UserControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldCreateUserSuccessfully() {
        // Given
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com", 25);
        
        // When
        ResponseEntity<UserDTO> response = restTemplate.postForEntity(
            "/api/users", request, UserDTO.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getName()).isEqualTo("John");
        
        // Verificar en BD
        Optional<User> savedUser = userRepository.findByEmail("john@example.com");
        assertThat(savedUser).isPresent();
        assertThat(savedUser.get().getName()).isEqualTo("John");
    }
}
```

### Test Slices
```java
// Web Layer Test
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void shouldReturnUser() throws Exception {
        // Given
        User user = new User(1L, "John", "john@example.com", 25);
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        // When & Then
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }
}

// Repository Test
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldFindByEmail() {
        // Given
        User user = new User("John", "john@example.com", 25);
        entityManager.persistAndFlush(user);
        
        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");
        
        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

## 8. Spring Boot Actuator

### Configuración
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

### Custom Health Indicator
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final UserRepository userRepository;
    
    @Override
    public Health health() {
        try {
            long userCount = userRepository.count();
            return Health.up()
                .withDetail("userCount", userCount)
                .withDetail("status", "Database is accessible")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Custom Metrics
```java
@Service
public class UserService {
    
    private final MeterRegistry meterRegistry;
    private final Counter userCreationCounter;
    private final Timer userCreationTimer;
    
    public UserService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.userCreationCounter = Counter.builder("user.created")
            .description("Number of users created")
            .register(meterRegistry);
        this.userCreationTimer = Timer.builder("user.creation.time")
            .description("Time taken to create user")
            .register(meterRegistry);
    }
    
    public User createUser(CreateUserRequest request) {
        return Timer.Sample.start(meterRegistry)
            .stop(userCreationTimer, () -> {
                User user = doCreateUser(request);
                userCreationCounter.increment();
                return user;
            });
    }
}
```

## 9. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre @Component, @Service, @Repository?
```java
// @Component - Genérico, bean gestionado por Spring
@Component
public class GenericComponent { }

// @Service - Lógica de negocio
@Service
public class UserService { }

// @Repository - Acceso a datos, translation de excepciones
@Repository
public class UserRepository { }

// @Controller - Capa de presentación web
@Controller
public class UserController { }

// @RestController - @Controller + @ResponseBody
@RestController
public class UserRestController { }
```

### ¿Qué es la auto-configuración?
Spring Boot analiza el classpath y configura automáticamente beans basado en:
- Dependencias presentes
- Propiedades de configuración
- Beans ya definidos

### ¿Cómo funciona @Transactional?
```java
// Propagación
@Transactional(propagation = Propagation.REQUIRED) // Default
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Transactional(propagation = Propagation.SUPPORTS)

// Aislamiento
@Transactional(isolation = Isolation.READ_COMMITTED)
@Transactional(isolation = Isolation.SERIALIZABLE)

// Rollback
@Transactional(rollbackFor = Exception.class)
@Transactional(noRollbackFor = BusinessException.class)
```

## 10. Mejores Prácticas

### Inyección de Dependencias
```java
// Preferir constructor injection
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### Configuración Externa
```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String version;
    private Database database = new Database();
    
    public static class Database {
        private String url;
        private int maxConnections = 10;
        // getters y setters
    }
}
```

### Perfiles de Entorno
```yaml
# Configuración base
spring:
  application:
    name: my-app

---
# Desarrollo
spring:
  config:
    activate:
      on-profile: dev
  h2:
    console:
      enabled: true

---
# Producción
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}
```

## Puntos Clave para Recordar

1. **Auto-configuración**: Spring Boot configura automáticamente basado en dependencias
2. **Starter dependencies**: Conjuntos preconfigurados de dependencias
3. **Profiles**: Configuración específica por entorno
4. **Actuator**: Monitoreo y métricas out-of-the-box
5. **Testing**: Test slices para testing eficiente
6. **Security**: Configuración declarativa con anotaciones
7. **Transacciones**: Gestión declarativa con @Transactional
8. **Inyección**: Preferir constructor injection
