# Kotlin - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### Kotlin vs Java
- **Interoperabilidad**: 100% compatible con Java
- **Null Safety**: Sistema de tipos que previene NPE
- **Concisión**: Menos código boilerplate
- **Funcional**: Soporte nativo para programación funcional

### Compilación
```kotlin
// Kotlin compila a bytecode Java
kotlinc MiClase.kt -include-runtime -d MiClase.jar
java -jar MiClase.jar
```

## 2. Null Safety

### Tipos Nullable vs Non-Nullable
```kotlin
// Non-nullable (por defecto)
var nombre: String = "Juan"
// nombre = null // Error de compilación

// Nullable
var apellido: String? = "Pérez"
apellido = null // OK
```

### Operadores de Null Safety
```kotlin
// Safe call operator
val longitud = apellido?.length

// Elvis operator
val resultado = apellido ?: "Sin apellido"

// Not-null assertion
val forzado = apellido!! // Lanza KotlinNullPointerException si es null

// Safe cast
val numero = valor as? Int
```

## 3. Propiedades y Backing Fields

### Propiedades con Getter/Setter
```kotlin
class Person {
    var name: String = ""
        get() = field.uppercase()
        set(value) {
            field = value.trim()
        }
    
    // Propiedad calculada
    val displayName: String
        get() = "Sr. $name"
        
    // Backing property
    private var _table: Map<String, Int>? = null
    val table: Map<String, Int>
        get() {
            if (_table == null) {
                _table = HashMap()
            }
            return _table ?: throw AssertionError("null")
        }
}
```

### Late-initialized Properties
```kotlin
class TestCase {
    lateinit var database: Database
    
    @Before
    fun setup() {
        database = Database() // Inicialización tardía
    }
    
    fun checkInitialized() {
        if (::database.isInitialized) {
            // Usar database
        }
    }
}
```

## 4. Clases y Objetos

### Data Classes
```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String
) {
    // Automáticamente genera: equals(), hashCode(), toString(), copy()
}

// Uso
val user = User(1, "Juan", "juan@email.com")
val userCopy = user.copy(name = "Pedro")
```

### Sealed Classes
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// Pattern matching exhaustivo
fun handleResult(result: Result<String>) = when (result) {
    is Result.Success -> println("Data: ${result.data}")
    is Result.Error -> println("Error: ${result.exception}")
    Result.Loading -> println("Loading...")
    // No necesita else, es exhaustivo
}
```

### Object Declarations (Singleton)
```kotlin
object DatabaseManager {
    fun connect() {
        println("Conectando a la base de datos")
    }
}

// Uso
DatabaseManager.connect()
```

### Companion Objects
```kotlin
class User {
    companion object {
        const val MAX_AGE = 120
        
        fun create(name: String): User {
            return User(name)
        }
    }
    
    private val name: String
    
    constructor(name: String) {
        this.name = name
    }
}

// Uso
val user = User.create("Juan")
```

## 5. Funciones

### Función de Extensión
```kotlin
// Extiende String con nueva funcionalidad
fun String.isValidEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

// Uso
val email = "test@example.com"
println(email.isValidEmail()) // true
```

### Funciones de Orden Superior
```kotlin
// Función que recibe otra función
fun operacion(a: Int, b: Int, op: (Int, Int) -> Int): Int {
    return op(a, b)
}

// Uso
val suma = operacion(5, 3) { x, y -> x + y }
val multiplicacion = operacion(5, 3) { x, y -> x * y }
```

### Lambdas y Referencias a Funciones
```kotlin
// Lambda
val suma: (Int, Int) -> Int = { a, b -> a + b }

// Función nombrada
fun multiplicar(a: Int, b: Int) = a * b

// Referencia a función
val operacion: (Int, Int) -> Int = ::multiplicar
```

### Infix Functions
```kotlin
infix fun Int.times(str: String) = str.repeat(this)

// Uso
println(3 times "Ho ") // "Ho Ho Ho "
```

## 6. Colecciones

### Colecciones Inmutables vs Mutables
```kotlin
// Inmutables (por defecto)
val lista = listOf(1, 2, 3)
val mapa = mapOf("a" to 1, "b" to 2)

// Mutables
val listaMutable = mutableListOf(1, 2, 3)
val mapaMutable = mutableMapOf("a" to 1, "b" to 2)
```

### Operaciones Funcionales
```kotlin
val numeros = listOf(1, 2, 3, 4, 5)

// Transformaciones
val dobles = numeros.map { it * 2 }
val pares = numeros.filter { it % 2 == 0 }
val suma = numeros.reduce { acc, n -> acc + n }
val sumaTodos = numeros.fold(0) { acc, n -> acc + n }

// Agrupación
val personas = listOf(
    Person("Juan", 25),
    Person("Ana", 30),
    Person("Pedro", 25)
)
val porEdad = personas.groupBy { it.age }
```

## 7. Corrutinas

### Conceptos Básicos
```kotlin
import kotlinx.coroutines.*

// Crear corrutina
fun main() = runBlocking {
    launch {
        delay(1000)
        println("Mundo!")
    }
    println("Hola ")
}
```

### Suspending Functions
```kotlin
suspend fun fetchUser(id: Long): User {
    delay(1000) // Simula llamada asíncrona
    return User(id, "Usuario $id")
}

// Uso
suspend fun example() {
    val user = fetchUser(1) // No bloquea el hilo
    println(user.name)
}
```

### Contexts y Dispatchers
```kotlin
// Diferentes contextos de ejecución
launch(Dispatchers.Main) {
    // UI thread
    updateUI()
}

launch(Dispatchers.IO) {
    // I/O operations
    val data = fetchFromNetwork()
}

launch(Dispatchers.Default) {
    // CPU-intensive work
    val result = heavyComputation()
}
```

### async/await
```kotlin
suspend fun fetchUserData(userId: Long): UserData = coroutineScope {
    // Ejecutar en paralelo
    val userDeferred = async { fetchUser(userId) }
    val postsDeferred = async { fetchPosts(userId) }
    
    // Esperar ambos resultados
    val user = userDeferred.await()
    val posts = postsDeferred.await()
    
    UserData(user, posts)
}
```

### Flow
```kotlin
// Crear Flow
fun numbersFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)
    }
}

// Consumir Flow
suspend fun example() {
    numbersFlow()
        .map { it * 2 }
        .filter { it > 4 }
        .collect { value ->
            println(value)
        }
}
```

## 8. Interoperabilidad con Java

### Llamar Java desde Kotlin
```kotlin
// Java
public class JavaClass {
    public static void staticMethod() { }
    public void instanceMethod() { }
}

// Kotlin
fun useJava() {
    JavaClass.staticMethod()
    val instance = JavaClass()
    instance.instanceMethod()
}
```

### Anotaciones para Java
```kotlin
class KotlinClass {
    @JvmStatic
    fun staticFromJava() {
        // Accesible como método estático desde Java
    }
    
    @JvmField
    val field = "Accesible como field desde Java"
    
    @JvmOverloads
    fun method(param1: String, param2: Int = 0) {
        // Genera sobrecargas para Java
    }
}
```

## 9. Delegación

### Delegación de Clases
```kotlin
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() { println(x) }
}

// Delegación
class Derived(b: Base) : Base by b {
    // Automáticamente implementa Base delegando a b
}
```

### Delegación de Propiedades
```kotlin
class Example {
    // Lazy initialization
    val lazyValue: String by lazy {
        println("Computed!")
        "Hello"
    }
    
    // Observable property
    var name: String by Delegates.observable("<no name>") { 
        prop, old, new -> println("$old -> $new")
    }
    
    // Delegación a Map
    val map = mapOf("name" to "John", "age" to 25)
    val name: String by map
    val age: Int by map
}
```

## 10. Scope Functions

### let, run, with, apply, also
```kotlin
val person = Person("Juan", 25)

// let - transformación con null safety
person.email?.let { email ->
    sendEmail(email)
}

// run - ejecutar bloque con contexto
val result = person.run {
    println("Name: $name")
    println("Age: $age")
    "Result"
}

// with - operaciones múltiples en objeto
with(person) {
    println("Name: $name")
    println("Age: $age")
}

// apply - configuración de objeto
val newPerson = Person().apply {
    name = "Ana"
    age = 30
}

// also - operaciones adicionales
val list = mutableListOf(1, 2, 3).also { 
    println("List created: $it")
}
```

## 11. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre `val` y `var`?
```kotlin
val inmutable = "No se puede cambiar"
// inmutable = "nuevo valor" // Error

var mutable = "Se puede cambiar"
mutable = "nuevo valor" // OK
```

### ¿Cómo funciona el null safety?
```kotlin
// El compilador previene NPE en tiempo de compilación
var nullable: String? = null
// nullable.length // Error de compilación

// Operadores seguros
val length = nullable?.length ?: 0
```

### ¿Qué son las corrutinas y por qué son útiles?
```kotlin
// Concurrencia sin bloqueo de hilos
suspend fun fetchData(): Data {
    // Suspende la corrutina, no el hilo
    delay(1000)
    return Data()
}
```

### ¿Diferencia entre `object` y `class`?
```kotlin
// object - Singleton
object MySingleton {
    fun doSomething() { }
}

// class - Instanciable
class MyClass {
    fun doSomething() { }
}
```

## 12. Mejores Prácticas

### Uso de Data Classes
```kotlin
// Para objetos que solo contienen datos
data class User(val id: Long, val name: String)

// No para clases con lógica compleja
class UserService {
    fun findUser(id: Long): User = TODO()
}
```

### Evitar !!
```kotlin
// Malo
val length = text!!.length

// Bueno
val length = text?.length ?: 0
```

### Usar Sealed Classes para Estados
```kotlin
sealed class ViewState {
    object Loading : ViewState()
    data class Content(val data: List<Item>) : ViewState()
    data class Error(val message: String) : ViewState()
}
```

## Puntos Clave para Recordar

1. **Null Safety**: Sistema de tipos que previene NPE
2. **Corrutinas**: Concurrencia ligera y no bloqueante
3. **Interoperabilidad**: 100% compatible con Java
4. **Funcional**: Soporte nativo para programación funcional
5. **Concisión**: Menos código boilerplate que Java
6. **Inmutabilidad**: Colecciones inmutables por defecto
7. **Extensiones**: Añadir funcionalidad sin herencia
8. **Delegación**: Patrón de diseño integrado en el lenguaje
