# Kotlin - Guía Completa Senior Backend

## 1. Tipos de Clases

### Clases Básicas
```kotlin
// Final por defecto (no heredable)
class Usuario(val nombre: String)

// Open - Heredable
open class Animal(val nombre: String) {
    open fun hacerSonido() = "Sonido genérico"
}

// Abstract - No instanciable
abstract class Vehiculo {
    abstract val ruedas: Int
    abstract fun acelerar()
    
    // Método concreto en clase abstracta
    fun mostrarInfo() = "Vehículo con $ruedas ruedas"
}

// Final explícito (redundante pero posible)
final class Documento
```

### Data Classes
```kotlin
data class Usuario(
    val id: Long,
    val nombre: String,
    val email: String,
    val activo: Boolean = true
) {
    // Genera automáticamente: equals(), hashCode(), toString(), copy(), componentN()
    
    // Funciones adicionales permitidas
    fun estaActivo() = activo
}

// Uso de copy() y destructuring
val usuario = Usuario(1L, "Juan", "juan@test.com")
val usuarioInactivo = usuario.copy(activo = false)
val (id, nombre, email) = usuario
```

### Value Classes (Inline Classes)
```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Email inválido" }
    }
}

// Sin overhead en runtime, solo wrapper en compile-time
fun buscarUsuario(id: UserId): Usuario? = null
fun enviarEmail(email: Email) = Unit
```

### Sealed Classes
```kotlin
sealed class Resultado<out T> {
    data class Exito<T>(val datos: T) : Resultado<T>()
    data class Error(val mensaje: String, val causa: Throwable? = null) : Resultado<Nothing>()
    object Cargando : Resultado<Nothing>()
    object Vacio : Resultado<Nothing>()
}

// Pattern matching exhaustivo
fun manejarResultado(resultado: Resultado<String>) = when (resultado) {
    is Resultado.Exito -> println("Datos: ${resultado.datos}")
    is Resultado.Error -> println("Error: ${resultado.mensaje}")
    Resultado.Cargando -> println("Cargando...")
    Resultado.Vacio -> println("Sin datos")
    // No necesita else - compilador garantiza exhaustividad
}

// Sealed interfaces (Kotlin 1.5+)
sealed interface Estado {
    object Inicial : Estado
    object Procesando : Estado
    data class Completado(val resultado: String) : Estado
}
```

### Enum Classes
```kotlin
enum class TipoUsuario(val codigo: String, val nivel: Int) {
    ADMIN("ADM", 3),
    EDITOR("EDT", 2),
    VIEWER("VWR", 1);
    
    fun puedeEditar() = nivel >= 2
    
    companion object {
        fun desdeCodigo(codigo: String) = values().find { it.codigo == codigo }
    }
}

// Enum con cuando
fun obtenerPermisos(tipo: TipoUsuario) = when (tipo) {
    TipoUsuario.ADMIN -> listOf("leer", "escribir", "eliminar")
    TipoUsuario.EDITOR -> listOf("leer", "escribir")
    TipoUsuario.VIEWER -> listOf("leer")
}
```

### Object Declarations
```kotlin
// Singleton thread-safe
object ConfiguracionBD {
    const val URL_DEFAULT = "jdbc:h2:mem:test"
    private val propiedades = mutableMapOf<String, String>()
    
    fun configurar(url: String, usuario: String) {
        propiedades["url"] = url
        propiedades["usuario"] = usuario
    }
}

// Object expression (clase anónima)
val comparadorPorNombre = object : Comparator<Usuario> {
    override fun compare(o1: Usuario, o2: Usuario) = o1.nombre.compareTo(o2.nombre)
}
```

## 2. Modificadores de Visibilidad

```kotlin
class EjemploVisibilidad {
    // Public por defecto
    val publico = "Visible desde cualquier lugar"
    
    // Private - solo en esta clase
    private val privado = "Solo en EjemploVisibilidad"
    
    // Protected - esta clase y subclases
    protected val protegido = "Visible en herencia"
    
    // Internal - mismo módulo
    internal val interno = "Visible en el módulo"
    
    private fun metodoPrivado() = "Solo aquí"
    protected open fun metodoProtegido() = "En herencia"
    internal fun metodoInterno() = "En el módulo"
}

// Top-level (a nivel de archivo)
private fun funcionPrivadaArchivo() = "Solo en este archivo"
internal fun funcionInternaModulo() = "En todo el módulo"
```

## 3. Propiedades y Campos

### Tipos de Propiedades
```kotlin
class PropiedadesEjemplo {
    // Val - inmutable (solo getter)
    val inmutable: String = "No cambia"
    
    // Var - mutable (getter y setter)
    var mutable: String = "Puede cambiar"
    
    // Const - constante en tiempo de compilación (solo en object o top-level)
    companion object {
        const val CONSTANTE = "Valor fijo"
    }
    
    // Lateinit - inicialización tardía (solo var, no primitivos)
    lateinit var configuracion: Configuracion
    
    // Lazy - inicialización perezosa thread-safe
    val costoso: String by lazy {
        println("Calculando...")
        "Valor calculado una sola vez"
    }
    
    // Custom getter/setter
    var nombre: String = ""
        get() = field.uppercase()
        set(value) {
            require(value.isNotBlank()) { "Nombre no puede estar vacío" }
            field = value.trim()
        }
    
    // Propiedad calculada (sin backing field)
    val nombreCompleto: String
        get() = "$nombre $apellido"
    
    private var apellido: String = ""
    
    // Backing property pattern
    private var _items: MutableList<String>? = null
    val items: List<String>
        get() {
            if (_items == null) _items = mutableListOf()
            return _items!!
        }
}

class Configuracion
```

### Delegación de Propiedades
```kotlin
class DelegacionEjemplo {
    // Lazy
    val perezoso: String by lazy { "Inicializado cuando se necesita" }
    
    // Observable
    var observado: String by Delegates.observable("inicial") { prop, old, new ->
        println("$old -> $new")
    }
    
    // Vetoable
    var validado: Int by Delegates.vetoable(0) { prop, old, new ->
        new >= 0 // Solo acepta valores positivos
    }
    
    // NotNull (para lateinit de primitivos)
    var primitivo: Int by Delegates.notNull()
    
    // Map delegation
    private val map = mapOf("nombre" to "Juan", "edad" to 30)
    val nombre: String by map
    val edad: Int by map
    
    // Custom delegate
    var custom: String by LoggingDelegate()
}

class LoggingDelegate {
    private var value: String = ""
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("Leyendo ${property.name}")
        return value
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("Escribiendo ${property.name} = $value")
        this.value = value
    }
}
```

## 4. Herencia y Polimorfismo

### Herencia Básica
```kotlin
// Clase base open
open class Animal(val nombre: String) {
    open val sonido: String = "..."
    open fun hacerSonido() = println(sonido)
    
    // Final - no se puede sobrescribir
    final fun dormir() = println("$nombre está durmiendo")
}

// Herencia con constructor primario
class Perro(nombre: String, val raza: String) : Animal(nombre) {
    override val sonido = "Guau"
    
    override fun hacerSonido() {
        super.hacerSonido() // Llamar al padre
        println("$nombre de raza $raza está ladrando")
    }
    
    // Método específico
    fun buscarHueso() = println("$nombre está buscando un hueso")
}

// Constructor secundario
class Gato : Animal {
    val vidas: Int
    
    constructor(nombre: String, vidas: Int = 9) : super(nombre) {
        this.vidas = vidas
    }
    
    override val sonido = "Miau"
}
```

### Interfaces
```kotlin
interface Volador {
    val alturaMaxima: Int
    
    fun volar() {
        println("Volando hasta $alturaMaxima metros")
    }
    
    // Propiedad abstracta
    val velocidadVuelo: Int
}

interface Nadador {
    fun nadar() = println("Nadando...")
}

// Implementación múltiple
class Pato(override val alturaMaxima: Int = 100) : Animal("Pato"), Volador, Nadador {
    override val sonido = "Cuac"
    override val velocidadVuelo = 50
    
    override fun volar() {
        super<Volador>.volar() // Especificar cual super usar
        println("El pato vuela bajo")
    }
}
```

## 5. Null Safety y Operadores

```kotlin
fun ejemplosNullSafety() {
    val texto: String? = obtenerTexto()
    
    // Safe call
    val longitud = texto?.length
    
    // Elvis operator
    val longitudOCero = texto?.length ?: 0
    
    // Safe cast
    val numero = texto as? Int
    
    // Not-null assertion (usar con cuidado)
    val longitudForzada = texto!!.length // KotlinNullPointerException si es null
    
    // Let para null safety
    texto?.let { textoNoNulo ->
        println("Texto: $textoNoNulo")
        println("Longitud: ${textoNoNulo.length}")
    }
    
    // If no null
    if (texto != null) {
        // Smart cast - compilador sabe que no es null
        println(texto.length)
    }
    
    // When con null
    when (texto) {
        null -> println("Es null")
        "" -> println("Está vacío")
        else -> println("Tiene contenido: $texto")
    }
}

fun obtenerTexto(): String? = if (Math.random() > 0.5) "texto" else null
```

## 6. Funciones Avanzadas

### Funciones de Extensión
```kotlin
// Extensión básica
fun String.esEmail(): Boolean = this.contains("@") && this.contains(".")

// Extensión con receptor nullable
fun String?.estaVacia(): Boolean = this == null || this.isEmpty()

// Extensión en clases genéricas
fun <T> List<T>.segundoONull(): T? = if (size >= 2) this[1] else null

// Extensión de propiedad
val String.primerCaracter: Char?
    get() = if (isEmpty()) null else this[0]

// Uso
fun ejemplosExtension() {
    println("test@email.com".esEmail()) // true
    println(null.estaVacia()) // true
    println(listOf(1, 2, 3).segundoONull()) // 2
    println("Hola".primerCaracter) // 'H'
}
```

### Scope Functions
```kotlin
data class Usuario(var nombre: String = "", var email: String = "")

fun ejemploScopeFunctions() {
    val usuario = Usuario()
    
    // let - transformación, null safety
    val nombreMayuscula = usuario.nombre.takeIf { it.isNotEmpty() }?.let { 
        it.uppercase() 
    }
    
    // run - ejecutar bloque con contexto objeto
    val resultado = usuario.run {
        nombre = "Juan"
        email = "juan@test.com"
        "Usuario creado: $nombre" // valor de retorno
    }
    
    // with - múltiples operaciones en objeto
    with(usuario) {
        println("Nombre: $nombre")
        println("Email: $email")
        // No retorna el objeto
    }
    
    // apply - configuración fluida
    val usuarioConfigurado = Usuario().apply {
        nombre = "Ana"
        email = "ana@test.com"
    } // retorna el objeto modificado
    
    // also - operaciones adicionales
    val lista = mutableListOf(1, 2, 3).also { 
        println("Lista creada con ${it.size} elementos")
    } // retorna el objeto original
    
    // takeIf / takeUnless - filtros inline
    val nombreValido = usuario.nombre.takeIf { it.length > 2 }
    val emailInvalido = usuario.email.takeUnless { it.contains("@") }
}
```

## 7. Colecciones y Operaciones Funcionales

### Tipos de Colecciones
```kotlin
fun ejemplosColecciones() {
    // Inmutables (por defecto)
    val lista = listOf(1, 2, 3)
    val conjunto = setOf("a", "b", "c")
    val mapa = mapOf("clave" to "valor", "numero" to 42)
    
    // Mutables
    val listaMutable = mutableListOf(1, 2, 3)
    val conjuntoMutable = mutableSetOf("a", "b")
    val mapaMutable = mutableMapOf("clave" to "valor")
    
    // Ranges
    val rango = 1..10
    val rangoExclusivo = 1 until 10
    val rangoDescendente = 10 downTo 1
    val rangoPaso = 1..10 step 2
}
```

### Operaciones Funcionales
```kotlin
data class Producto(val nombre: String, val precio: Double, val categoria: String)

fun operacionesColecciones() {
    val productos = listOf(
        Producto("Laptop", 1000.0, "Electrónicos"),
        Producto("Mesa", 200.0, "Muebles"),
        Producto("Teléfono", 500.0, "Electrónicos"),
        Producto("Silla", 150.0, "Muebles")
    )
    
    // Transformaciones
    val nombres = productos.map { it.nombre }
    val precios = productos.map(Producto::precio)
    
    // Filtros
    val electronicos = productos.filter { it.categoria == "Electrónicos" }
    val caros = productos.filter { it.precio > 300 }
    
    // Agrupación
    val porCategoria = productos.groupBy { it.categoria }
    
    // Agregaciones
    val precioTotal = productos.sumOf { it.precio }
    val precioPromedio = productos.map { it.precio }.average()
    val masCarо = productos.maxByOrNull { it.precio }
    
    // Fold/Reduce
    val resumen = productos.fold("Productos: ") { acc, producto ->
        "$acc${producto.nombre} "
    }
    
    // Partición
    val (costosos, baratos) = productos.partition { it.precio > 300 }
    
    // FlatMap
    val todasLasLetras = productos
        .flatMap { it.nombre.toList() }
        .distinct()
}
```

## 8. When Expression Avanzado

```kotlin
fun ejemplosWhen() {
    val obj: Any = "Hola"
    
    // When con tipos
    val resultado = when (obj) {
        is String -> "Es texto: ${obj.length} caracteres"
        is Int -> "Es número: ${obj * 2}"
        is List<*> -> "Es lista con ${obj.size} elementos"
        else -> "Tipo desconocido"
    }
    
    // When con rangos
    val edad = 25
    val categoria = when (edad) {
        in 0..12 -> "Niño"
        in 13..17 -> "Adolescente"
        in 18..64 -> "Adulto"
        in 65..120 -> "Adulto mayor"
        else -> "Edad inválida"
    }
    
    // When con condiciones múltiples
    val numero = 15
    val descripcion = when {
        numero % 15 == 0 -> "Múltiplo de 15"
        numero % 3 == 0 -> "Múltiplo de 3"
        numero % 5 == 0 -> "Múltiplo de 5"
        numero < 0 -> "Negativo"
        else -> "Número normal"
    }
}
```

## 9. Manejo de Errores

### Result Type Manual
```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
    
    inline fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> Success(transform(value))
        is Failure -> this
    }
    
    inline fun <R> flatMap(transform: (T) -> Result<R>): Result<R> = when (this) {
        is Success -> transform(value)
        is Failure -> this
    }
    
    fun getOrNull(): T? = when (this) {
        is Success -> value
        is Failure -> null
    }
    
    fun getOrElse(default: T): T = when (this) {
        is Success -> value
        is Failure -> default
    }
    
    fun getOrThrow(): T = when (this) {
        is Success -> value
        is Failure -> throw error
    }
    
    val isSuccess: Boolean get() = this is Success
    val isFailure: Boolean get() = this is Failure
    
    fun exceptionOrNull(): Throwable? = when (this) {
        is Success -> null
        is Failure -> error
    }
}

// Funciones de utilidad para Result manual
fun <T> Result<T>.onSuccess(action: (T) -> Unit): Result<T> {
    if (this is Result.Success) action(value)
    return this
}

fun <T> Result<T>.onFailure(action: (Throwable) -> Unit): Result<T> {
    if (this is Result.Failure) action(error)
    return this
}

// Extension function para fold en Result manual
fun <T, R> Result<T>.fold(
    onSuccess: (T) -> R,
    onFailure: (Throwable) -> R
): R = when (this) {
    is Result.Success -> onSuccess(value)
    is Result.Failure -> onFailure(error)
}

// Extension para getOrElse en Result manual
fun <T> Result<T>.getOrElse(defaultValue: (Throwable) -> T): T = when (this) {
    is Result.Success -> value
    is Result.Failure -> defaultValue(error)
}

// Extension para mapCatching en Result manual
fun <T, R> Result<T>.mapCatching(transform: (T) -> R): Result<R> = when (this) {
    is Result.Success -> try {
        Result.Success(transform(value))
    } catch (e: Exception) {
        Result.Failure(e)
    }
    is Result.Failure -> this
}
```

### RunCatching y Result Stdlib
```kotlin
fun ejemploRunCatching() {
    // runCatching convierte excepciones a Result
    val resultado = runCatching {
        "123".toInt() * 2
    }
    
    // onSuccess/onFailure - ejecutar side effects
    resultado
        .onSuccess { println("Resultado: $it") }
        .onFailure { println("Error: ${it.message}") }
    
    // fold - transformar ambos casos en un solo tipo
    val mensaje = runCatching {
        "abc".toInt()
    }.fold(
        onSuccess = { value -> "Éxito: $value" },
        onFailure = { error -> "Error: ${error.message}" }
    )
    println(mensaje) // "Error: For input string: \"abc\""
    
    // getOrElse - valor por defecto en caso de error
    val valorSeguro = runCatching {
        "invalid".toInt()
    }.getOrElse { 0 }
    
    // getOrDefault - similar a getOrElse pero más directo
    val conDefault = runCatching {
        "invalid".toInt()
    }.getOrDefault(42)
    
    // getOrNull - null si hay error
    val nullable = runCatching {
        "invalid".toInt()
    }.getOrNull() // null
    
    // getOrThrow - relanza la excepción
    val valor = runCatching {
        "123".toInt()
    }.getOrThrow() // 123, o lanza excepción si falló
    
    // isSuccess/isFailure - verificar estado
    val esExitoso = resultado.isSuccess
    val haFallado = resultado.isFailure
    
    // exceptionOrNull - obtener la excepción si existe
    val excepcion = resultado.exceptionOrNull()
}

// Encadenamiento con mapCatching
fun ejemploEncadenamiento() {
    val resultado = runCatching { "123" }
        .mapCatching { it.toInt() }           // String -> Int
        .mapCatching { it * 2 }               // Int -> Int
        .mapCatching { "Resultado: $it" }     // Int -> String
        .fold(
            onSuccess = { it },
            onFailure = { "Error en el procesamiento" }
        )
    
    println(resultado) // "Resultado: 246"
}

// Recover - recuperarse de errores
fun ejemploRecover() {
    val resultado = runCatching {
        "abc".toInt() // Fallará
    }.recover { throwable ->
        when (throwable) {
            is NumberFormatException -> 0
            else -> throw throwable // Re-lanzar otros errores
        }
    }.getOrThrow()
    
    println(resultado) // 0
    
    // recoverCatching - recover que también puede fallar
    val conRecoverCatching = runCatching {
        "abc".toInt()
    }.recoverCatching { error ->
        // Intento de recuperación que también puede fallar
        "42".toInt() // Esta vez funciona
    }.getOrDefault(-1)
    
    println(conRecoverCatching) // 42
}

// Validaciones con require/check/error
class UsuarioService {
    fun crearUsuario(nombre: String, edad: Int): Result<Usuario> {
        return runCatching {
            // require - validar parámetros (IllegalArgumentException)
            require(nombre.isNotBlank()) { "El nombre no puede estar vacío" }
            require(edad >= 0) { "La edad debe ser positiva" }
            require(edad <= 150) { "Edad inválida" }
            
            val usuario = Usuario(1L, nombre, "email@test.com")
            
            // check - validar estado (IllegalStateException)
            check(usuario.nombre.isNotEmpty()) { "Usuario creado en estado inválido" }
            
            usuario
        }.fold(
            onSuccess = { Result.success(it) },
            onFailure = { error ->
                when (error) {
                    is IllegalArgumentException -> Result.failure(error)
                    is IllegalStateException -> Result.failure(error)
                    else -> Result.failure(RuntimeException("Error inesperado", error))
                }
            }
        )
    }
    
    // Ejemplo usando diferentes métodos de Result
    fun procesarUsuario(id: Long): String {
        return buscarUsuario(id)
            .mapCatching { usuario ->
                requireNotNull(usuario.email) { "Usuario sin email" }
                "Procesando: ${usuario.nombre} (${usuario.email})"
            }
            .recover { error ->
                "Error procesando usuario $id: ${error.message}"
            }
            .getOrDefault("Usuario no encontrado")
    }
    
    private fun buscarUsuario(id: Long): Result<Usuario> {
        return if (id > 0) {
            Result.success(Usuario(id, "Usuario $id", "user$id@test.com"))
        } else {
            Result.failure(IllegalArgumentException("ID inválido"))
        }
    }
}

// Combinando múltiples operaciones que pueden fallar
fun ejemploCombinado() {
    data class DatosCompletos(val usuario: Usuario, val configuracion: String, val permisos: List<String>)
    
    fun cargarDatosCompletos(userId: Long): Result<DatosCompletos> {
        return runCatching {
            val usuario = cargarUsuario(userId).getOrThrow()
            val config = cargarConfiguracion(userId).getOrThrow()
            val permisos = cargarPermisos(userId).getOrThrow()
            
            DatosCompletos(usuario, config, permisos)
        }
    }
    
    // Versión más robusta con manejo individual
    fun cargarDatosCompletosRobusto(userId: Long): Result<DatosCompletos> {
        val usuarioResult = cargarUsuario(userId)
        val configResult = cargarConfiguracion(userId)
        val permisosResult = cargarPermisos(userId)
        
        return when {
            usuarioResult.isFailure -> usuarioResult.map { /* nunca se ejecuta */ DatosCompletos(it, "", emptyList()) }
            configResult.isFailure -> configResult.map { /* nunca se ejecuta */ DatosCompletos(Usuario(0, "", ""), it, emptyList()) }
            permisosResult.isFailure -> permisosResult.map { /* nunca se ejecuta */ DatosCompletos(Usuario(0, "", ""), "", it) }
            else -> Result.success(
                DatosCompletos(
                    usuarioResult.getOrThrow(),
                    configResult.getOrThrow(),
                    permisosResult.getOrThrow()
                )
            )
        }
    }
    
    private fun cargarUsuario(id: Long): Result<Usuario> = runCatching {
        Usuario(id, "Usuario $id", "user$id@test.com")
    }
    
    private fun cargarConfiguracion(id: Long): Result<String> = runCatching {
        "config-$id"
    }
    
    private fun cargarPermisos(id: Long): Result<List<String>> = runCatching {
        listOf("read", "write")
    }
}
```

## 10. Corrutinas

### Conceptos Básicos
```kotlin
import kotlinx.coroutines.*

// Función suspendible
suspend fun operacionLenta(): String {
    delay(1000) // Suspende la corrutina, no bloquea el hilo
    return "Resultado"
}

// Crear y ejecutar corrutinas
fun ejemploBasico() {
    runBlocking {
        println("Inicio")
        val resultado = operacionLenta()
        println("Fin: $resultado")
    }
}
```

### Constructores de Corrutinas
```kotlin
fun ejemploConstructores() = runBlocking {
    // launch - fire and forget, retorna Job
    val job = launch {
        repeat(3) {
            println("Trabajo $it")
            delay(500)
        }
    }
    
    // async - retorna Deferred<T>
    val deferred = async {
        delay(1000)
        "Resultado async"
    }
    
    // Esperar resultados
    job.join() // Esperar que complete
    val resultado = deferred.await() // Obtener resultado
    println(resultado)
}
```

### Flow - Streams Reactivos
```kotlin
fun ejemploFlow() = runBlocking {
    // Crear Flow
    val numerosFlow = flow {
        for (i in 1..5) {
            delay(100)
            emit(i)
        }
    }
    
    // Consumir Flow
    numerosFlow
        .map { it * 2 }
        .filter { it > 4 }
        .collect { valor ->
            println("Recibido: $valor")
        }
}
```

## 11. Interoperabilidad con Java

### Anotaciones JVM
```kotlin
class InteropJava {
    // @JvmStatic - método estático en Java
    companion object {
        @JvmStatic
        fun metodoEstatico() = "Accesible como static desde Java"
    }
    
    // @JvmField - campo público directo
    @JvmField
    val campoPublico = "Acceso directo desde Java"
    
    // @JvmOverloads - genera sobrecargas para parámetros por defecto
    @JvmOverloads
    fun metodoConDefecto(param1: String, param2: Int = 0, param3: Boolean = true) {
        println("$param1, $param2, $param3")
    }
}
```

## 12. Patrones de Diseño

### Singleton
```kotlin
// Object declaration - thread-safe singleton
object DatabaseManager {
    private var connection: String? = null
    
    fun getConnection(): String {
        if (connection == null) {
            connection = "Database Connection"
        }
        return connection!!
    }
}

// Singleton con by lazy (thread-safe)
class ConfigManager private constructor() {
    companion object {
        val instance: ConfigManager by lazy { ConfigManager() }
    }
    
    fun getConfig(key: String): String = "valor-$key"
}
```

### Factory Pattern
```kotlin
sealed class Animal {
    abstract fun makeSound(): String
}

class Perro : Animal() {
    override fun makeSound() = "Guau"
}

class Gato : Animal() {
    override fun makeSound() = "Miau"
}

// Factory con when
object AnimalFactory {
    fun crear(tipo: String): Animal = when (tipo.lowercase()) {
        "perro" -> Perro()
        "gato" -> Gato()
        else -> throw IllegalArgumentException("Tipo de animal desconocido: $tipo")
    }
}
```

## 13. Testing

### Testing Básico con JUnit 5
```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.DisplayName

class CalculadoraTest {
    private lateinit var calculadora: Calculadora
    
    @BeforeEach
    fun setUp() {
        calculadora = Calculadora()
    }
    
    @Test
    @DisplayName("Debería sumar dos números positivos")
    fun `should add two positive numbers`() {
        // Given
        val a = 5
        val b = 3
        
        // When
        val resultado = calculadora.sumar(a, b)
        
        // Then
        assertEquals(8, resultado)
    }
    
    @Test
    fun `should throw exception when dividing by zero`() {
        assertThrows<ArithmeticException> {
            calculadora.dividir(10, 0)
        }
    }
}

class Calculadora {
    fun sumar(a: Int, b: Int) = a + b
    fun dividir(a: Int, b: Int): Int {
        if (b == 0) throw ArithmeticException("División por cero")
        return a / b
    }
}
```

## 14. Spring Boot Integration

### Controllers REST
```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {
    
    @GetMapping
    fun getAllUsers(): ResponseEntity<List<UserDto>> {
        val users = userService.findAll()
        return ResponseEntity.ok(users)
    }
    
    @GetMapping("/{id}")
    fun getUserById(@PathVariable id: Long): ResponseEntity<UserDto> {
        return userService.findById(id)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()
    }
    
    @PostMapping
    fun createUser(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<UserDto> {
        val user = userService.create(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(user)
    }
}
```

### DTOs y Validación
```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    val name: String,
    
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email format is invalid")
    val email: String,
    
    @field:Min(value = 18, message = "Age must be at least 18")
    @field:Max(value = 120, message = "Age must be at most 120")
    val age: Int?
)

data class UserDto(
    val id: Long,
    val name: String,
    val email: String,
    val age: Int?,
    val active: Boolean
)
```

## 15. Mejores Prácticas

### Naming Conventions
```kotlin
// Clases: PascalCase
class UserService
data class CreateUserRequest
sealed class ApiResult

// Funciones y variables: camelCase
fun calculateTotalPrice(): Double
val userName: String
var isActive: Boolean

// Constantes: SCREAMING_SNAKE_CASE
companion object {
    const val MAX_RETRY_ATTEMPTS = 3
    const val DEFAULT_TIMEOUT_MS = 5000L
}
```

### Code Organization
```kotlin
// Estructura de paquetes recomendada:
/*
src/main/kotlin/
├── com/empresa/proyecto/
│   ├── domain/                     # Lógica de negocio
│   │   ├── model/                  # Entidades y Value Objects
│   │   ├── repository/             # Interfaces de repositorio
│   │   └── service/                # Servicios de dominio
│   ├── application/                # Casos de uso
│   │   ├── usecase/               # Use cases
│   │   └── dto/                   # DTOs de aplicación
│   ├── infrastructure/            # Detalles de implementación
│   │   ├── database/              # JPA, repositorios
│   │   ├── web/                   # Controllers, REST
│   │   └── config/                # Configuración
│   └── Application.kt             # Main class
*/
```

## 16. Resumen de Conceptos Clave

### Tipos de Clases
- **`class`**: Final por defecto, no heredable
- **`open class`**: Heredable, debe marcarse explícitamente
- **`abstract class`**: No instanciable, puede tener implementación parcial
- **`data class`**: Auto-genera equals, hashCode, toString, copy
- **`value class`**: Wrapper sin overhead en runtime
- **`sealed class`**: Jerarquía cerrada, pattern matching exhaustivo
- **`enum class`**: Conjunto fijo de constantes con comportamiento
- **`object`**: Singleton thread-safe

### Modificadores de Visibilidad
- **`public`**: Visible en todas partes (por defecto)
- **`private`**: Solo en la clase/archivo actual
- **`protected`**: Clase actual y subclases
- **`internal`**: Visible en el mismo módulo

### Propiedades
- **`val`**: Inmutable (solo getter)
- **`var`**: Mutable (getter y setter)
- **`const val`**: Constante en tiempo de compilación
- **`lateinit var`**: Inicialización tardía
- **`by lazy`**: Inicialización perezosa thread-safe

### Null Safety
- **`Type?`**: Tipo nullable
- **`?.`**: Safe call operator
- **`?:`**: Elvis operator
- **`!!`**: Not-null assertion (usar con cuidado)
- **`as?`**: Safe cast

### Funciones Especiales
- **Extension functions**: Añadir funcionalidad sin herencia
- **Infix functions**: Sintaxis como operadores
- **Inline functions**: Evitar overhead de lambdas
- **Suspend functions**: Para corrutinas

### Scope Functions
- **`let`**: Transformación con null safety
- **`run`**: Ejecutar con contexto
- **`with`**: Múltiples operaciones en objeto
- **`apply`**: Configuración fluida
- **`also`**: Operaciones adicionales

### Conceptos Avanzados
- **Corrutinas**: Concurrencia ligera y no bloqueante
- **Flow**: Streams reactivos
- **Delegación**: De clases y propiedades
- **DSLs**: Domain Specific Languages con lambdas
- **Result types**: Manejo funcional de errores

## Puntos Clave para Recordar

1. **Null Safety**: Sistema de tipos que previene NPE
2. **Corrutinas**: Concurrencia ligera y no bloqueante
3. **Interoperabilidad**: 100% compatible con Java
4. **Funcional**: Soporte nativo para programación funcional
5. **Concisión**: Menos código boilerplate que Java
6. **Inmutabilidad**: Colecciones inmutables por defecto
7. **Extensiones**: Añadir funcionalidad sin herencia
8. **Delegación**: Patrón de diseño integrado en el lenguaje
9. **Value Classes**: Wrappers sin overhead de performance
10. **Sealed Classes**: Perfect para modelar estados y resultados
