# TDD & Testing - Guía de Entrevista Senior Backend

## 1. Test-Driven Development (TDD)

### Ciclo Red-Green-Refactor
1. **Red**: Escribir un test que falle
2. **Green**: Escribir el mínimo código para que pase
3. **Refactor**: Mejorar el código manteniendo los tests verdes

### Ejemplo Práctico en Kotlin
```kotlin
// 1. RED - Test que falla
class CalculatorTest {
    
    @Test
    fun `should add two numbers correctly`() {
        // Given
        val calculator = Calculator()
        
        // When
        val result = calculator.add(2, 3)
        
        // Then
        assertThat(result).isEqualTo(5)
    }
}

// 2. GREEN - Mínimo código para pasar
class Calculator {
    fun add(a: Int, b: Int): Int {
        return 5 // Hard-coded para pasar el test
    }
}

// 3. Más tests para forzar implementación real
@Test
fun `should add different numbers correctly`() {
    val calculator = Calculator()
    
    assertThat(calculator.add(1, 1)).isEqualTo(2)
    assertThat(calculator.add(10, 5)).isEqualTo(15)
    assertThat(calculator.add(-1, 1)).isEqualTo(0)
}

// 4. REFACTOR - Implementación real
class Calculator {
    fun add(a: Int, b: Int): Int {
        return a + b
    }
}
```

## 2. Pirámide de Testing

### Unit Tests (Base de la pirámide)
```kotlin
// Tests rápidos, aislados, sin dependencias externas
class UserValidatorTest {
    
    private val userValidator = UserValidator()
    
    @Test
    fun `should validate user with correct data`() {
        // Given
        val user = User(
            name = "John Doe",
            email = "john@example.com",
            age = 25
        )
        
        // When
        val result = userValidator.validate(user)
        
        // Then
        assertThat(result.isValid).isTrue()
        assertThat(result.errors).isEmpty()
    }
    
    @Test
    fun `should return errors for invalid user`() {
        // Given
        val user = User(
            name = "",
            email = "invalid-email",
            age = 17
        )
        
        // When
        val result = userValidator.validate(user)
        
        // Then
        assertThat(result.isValid).isFalse()
        assertThat(result.errors).containsExactlyInAnyOrder(
            "Name cannot be empty",
            "Invalid email format",
            "Age must be at least 18"
        )
    }
}
```

### Integration Tests
```kotlin
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {
    
    @Container
    companion object {
        @JvmStatic
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
    }
    
    @Autowired
    lateinit var userRepository: UserRepository
    
    @Test
    fun `should save and retrieve user`() {
        // Given
        val user = User(
            name = "John Doe",
            email = "john@example.com",
            age = 25
        )
        
        // When
        val savedUser = userRepository.save(user)
        val retrievedUser = userRepository.findById(savedUser.id!!)
        
        // Then
        assertThat(retrievedUser).isPresent
        assertThat(retrievedUser.get().name).isEqualTo("John Doe")
        assertThat(retrievedUser.get().email).isEqualTo("john@example.com")
    }
}
```

## 3. Mocking y Test Doubles

### MockK (Kotlin-first mocking)
```kotlin
class UserServiceTest {
    
    private val userRepository = mockk<UserRepository>()
    private val emailService = mockk<EmailService>()
    private val userService = UserService(userRepository, emailService)
    
    @Test
    fun `should create user successfully`() {
        // Given
        val request = CreateUserRequest("John", "john@example.com", 25)
        val savedUser = User(1L, "John", "john@example.com", 25)
        
        every { userRepository.findByEmail("john@example.com") } returns Optional.empty()
        every { userRepository.save(any()) } returns savedUser
        every { emailService.sendWelcomeEmail(any()) } just Runs
        
        // When
        val result = userService.createUser(request)
        
        // Then
        assertThat(result.name).isEqualTo("John")
        
        verify { userRepository.save(any()) }
        verify { emailService.sendWelcomeEmail("john@example.com") }
    }
    
    @Test
    fun `should verify call order`() {
        // Given
        every { userRepository.findByEmail(any()) } returns Optional.empty()
        every { userRepository.save(any()) } returns mockk()
        every { emailService.sendWelcomeEmail(any()) } just Runs
        
        // When
        userService.createUser(request)
        
        // Then
        verifyOrder {
            userRepository.findByEmail(any())
            userRepository.save(any())
            emailService.sendWelcomeEmail(any())
        }
    }
}
```

## 4. Testcontainers

### Database Testing
```kotlin
@SpringBootTest
@Testcontainers
class DatabaseIntegrationTest {
    
    @Container
    companion object {
        @JvmStatic
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("init.sql")
    }
    
    @DynamicPropertySource
    companion object {
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
    
    @Autowired
    lateinit var userRepository: UserRepository
    
    @Test
    fun `should work with real database`() {
        val user = userRepository.save(User(name = "John", email = "john@example.com", age = 25))
        assertThat(user.id).isNotNull()
    }
}
```

## 5. Test Slices en Spring Boot

### @WebMvcTest
```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {
    
    @Autowired
    lateinit var mockMvc: MockMvc
    
    @MockBean
    lateinit var userService: UserService
    
    @Test
    fun `should return user when found`() {
        // Given
        val user = User(1L, "John", "john@example.com", 25)
        whenever(userService.findById(1L)).thenReturn(user)
        
        // When & Then
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"))
    }
}
```

### @DataJpaTest
```kotlin
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    lateinit var testEntityManager: TestEntityManager
    
    @Autowired
    lateinit var userRepository: UserRepository
    
    @Test
    fun `should find user by email`() {
        // Given
        val user = User(name = "John", email = "john@example.com", age = 25)
        testEntityManager.persistAndFlush(user)
        
        // When
        val foundUser = userRepository.findByEmail("john@example.com")
        
        // Then
        assertThat(foundUser).isPresent
        assertThat(foundUser.get().name).isEqualTo("John")
    }
}
```

## 6. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre TDD y BDD?
- **TDD**: Test-Driven Development, enfoque técnico desde desarrollador
- **BDD**: Behavior-Driven Development, enfoque desde comportamiento del usuario
- **TDD**: Given-When-Then en tests unitarios
- **BDD**: Scenarios escritos en lenguaje natural (Gherkin)

### ¿Cuándo usar mocks vs stubs?
- **Mocks**: Cuando necesitas verificar interacciones (llamadas a métodos)
- **Stubs**: Cuando solo necesitas retornar valores predefinidos
- **Regla**: Mock para verificación, Stub para estado

### ¿Qué es la pirámide de testing?
- **Base**: Muchos unit tests (rápidos, baratos)
- **Medio**: Algunos integration tests (moderados)
- **Cima**: Pocos E2E tests (lentos, caros)

## 7. Mejores Prácticas

### Test Data Builders
```kotlin
class UserTestDataBuilder {
    private var name: String = "Default Name"
    private var email: String = "default@example.com"
    private var age: Int = 25
    
    fun withName(name: String) = apply { this.name = name }
    fun withEmail(email: String) = apply { this.email = email }
    fun withAge(age: Int) = apply { this.age = age }
    
    fun build() = User(name = name, email = email, age = age)
}

// Uso en tests
@Test
fun `should validate user age`() {
    val user = UserTestDataBuilder()
        .withAge(17)
        .build()
        
    val result = userValidator.validate(user)
    
    assertThat(result.isValid).isFalse()
}
```

## Puntos Clave para Recordar

1. **TDD Cycle**: Red → Green → Refactor
2. **Test Pyramid**: Muchos unit tests, algunos integration, pocos E2E
3. **Test Doubles**: Dummy, Stub, Spy, Mock, Fake
4. **AAA Pattern**: Arrange (Given) → Act (When) → Assert (Then)
5. **FIRST Principles**: Fast, Independent, Repeatable, Self-validating, Timely
6. **Test Naming**: Descriptivo, específico, indica comportamiento esperado
7. **Testcontainers**: Para integration tests con dependencias reales
8. **Coverage**: Métrica útil pero no objetivo final
9. **Mocking**: Usar con moderación, preferir objetos reales cuando sea posible
10. **Test Slices**: @WebMvcTest, @DataJpaTest, @JsonTest para tests focalizados
