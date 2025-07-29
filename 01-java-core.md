# Java Core - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### JVM, JRE, JDK
- **JVM (Java Virtual Machine)**: Entorno de ejecución que interpreta bytecode
- **JRE (Java Runtime Environment)**: JVM + librerías estándar
- **JDK (Java Development Kit)**: JRE + herramientas de desarrollo (javac, jar, etc.)

### Compilación y Ejecución
```java
// Compilación: .java → .class (bytecode)
javac MiClase.java
// Ejecución: JVM interpreta bytecode
java MiClase
```

## 2. Tipos de Datos y Memoria

### Tipos Primitivos vs Objetos
```java
// Primitivos (stack)
int numero = 10;
boolean flag = true;

// Objetos (heap)
String texto = "Hola";
List<Integer> lista = new ArrayList<>();
```

### Autoboxing/Unboxing
```java
// Autoboxing: primitivo → wrapper
Integer obj = 5; // Integer.valueOf(5)

// Unboxing: wrapper → primitivo  
int valor = obj; // obj.intValue()
```

## 3. Gestión de Memoria

### Stack vs Heap
- **Stack**: Variables locales, referencias, métodos
- **Heap**: Objetos, arrays, variables de instancia

### Garbage Collection
```java
// Eligible for GC cuando no hay referencias
String str = new String("test");
str = null; // Objeto elegible para GC
```

**Tipos de GC**:
- **Serial GC**: Single-threaded
- **Parallel GC**: Multi-threaded
- **G1 GC**: Low-latency para heaps grandes
- **ZGC/Shenandoah**: Ultra-low latency

## 4. Colecciones

### List
```java
// ArrayList: Acceso rápido O(1), inserción O(n)
List<String> arrayList = new ArrayList<>();

// LinkedList: Inserción rápida O(1), acceso O(n)
List<String> linkedList = new LinkedList<>();
```

### Set
```java
// HashSet: O(1) promedio, no ordenado
Set<String> hashSet = new HashSet<>();

// TreeSet: O(log n), ordenado
Set<String> treeSet = new TreeSet<>();

// LinkedHashSet: O(1) + orden inserción
Set<String> linkedHashSet = new LinkedHashSet<>();
```

### Map
```java
// HashMap: O(1) promedio
Map<String, Integer> hashMap = new HashMap<>();

// TreeMap: O(log n), ordenado por clave
Map<String, Integer> treeMap = new TreeMap<>();

// ConcurrentHashMap: Thread-safe
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
```

## 5. Programación Funcional (Java 8+)

### Lambda Expressions
```java
// Sintaxis básica
(parametros) -> expresion
(parametros) -> { statements; }

// Ejemplos
Predicate<String> isEmpty = s -> s.isEmpty();
Function<String, Integer> length = s -> s.length();
Consumer<String> print = s -> System.out.println(s);
```

### Streams
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Operaciones intermedias (lazy)
numbers.stream()
    .filter(n -> n % 2 == 0)      // Filtrar pares
    .map(n -> n * 2)              // Multiplicar por 2
    .sorted()                     // Ordenar
    .collect(Collectors.toList()); // Terminal

// Operaciones terminales
Optional<Integer> max = numbers.stream().max(Integer::compareTo);
int sum = numbers.stream().mapToInt(Integer::intValue).sum();
```

## 6. Multithreading y Concurrencia

### Thread vs Runnable
```java
// Extendiendo Thread
class MiThread extends Thread {
    public void run() {
        System.out.println("Ejecutando thread");
    }
}

// Implementando Runnable (recomendado)
class MiTask implements Runnable {
    public void run() {
        System.out.println("Ejecutando tarea");
    }
}
```

### Synchronized
```java
// Método sincronizado
public synchronized void metodoSincronizado() {
    // Código thread-safe
}

// Bloque sincronizado
public void metodo() {
    synchronized(this) {
        // Código thread-safe
    }
}
```

### ExecutorService
```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Enviar tareas
executor.submit(() -> {
    System.out.println("Tarea ejecutada");
});

// Cerrar executor
executor.shutdown();
```

### CompletableFuture
```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "Hola")
    .thenApply(s -> s + " Mundo")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + "!"));

String result = future.get(); // "Hola Mundo!"
```

## 7. Manejo de Excepciones

### Jerarquía de Excepciones
```
Throwable
├── Error (OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException (unchecked)
    └── Checked Exceptions (IOException, SQLException)
```

### Try-with-resources
```java
// Automático cierre de recursos
try (FileReader file = new FileReader("archivo.txt");
     BufferedReader buffer = new BufferedReader(file)) {
    
    return buffer.readLine();
} // Automáticamente se cierra
```

## 8. Nuevas Características Java 11+

### Var (Java 10)
```java
var lista = new ArrayList<String>(); // Inferencia de tipos
var mapa = Map.of("key", "value");
```

### Records (Java 14+)
```java
// Inmutable data class
public record Person(String name, int age) {
    // Constructor, equals, hashCode, toString automáticos
}
```

### Pattern Matching (Java 17+)
```java
// Switch expressions
String resultado = switch (dia) {
    case "LUNES", "MARTES" -> "Inicio de semana";
    case "MIERCOLES" -> "Medio";
    default -> "Fin de semana";
};
```

## 9. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre == y equals()?
```java
String a = "hello";
String b = "hello";
String c = new String("hello");

a == b;        // true (mismo objeto en pool)
a == c;        // false (diferentes objetos)
a.equals(c);   // true (mismo contenido)
```

### ¿Qué es el contrato equals() y hashCode()?
```java
// Si dos objetos son equals(), deben tener el mismo hashCode()
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (obj == null || getClass() != obj.getClass()) return false;
    Person person = (Person) obj;
    return age == person.age && Objects.equals(name, person.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

### ¿Cuándo usar StringBuilder vs StringBuffer?
```java
// StringBuilder: Single-threaded (más rápido)
StringBuilder sb = new StringBuilder();
sb.append("Hello").append(" World");

// StringBuffer: Thread-safe (más lento)
StringBuffer sbf = new StringBuffer();
sbf.append("Hello").append(" World");
```

## 10. Mejores Prácticas

### Immutabilidad
```java
public final class ImmutablePerson {
    private final String name;
    private final List<String> hobbies;
    
    public ImmutablePerson(String name, List<String> hobbies) {
        this.name = name;
        this.hobbies = List.copyOf(hobbies); // Defensive copy
    }
    
    public List<String> getHobbies() {
        return hobbies; // Ya es inmutable
    }
}
```

### Uso de Optional
```java
// Evitar null pointer exceptions
Optional<String> nombre = Optional.ofNullable(persona.getNombre());

// Operaciones con Optional
String resultado = nombre
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .orElse("DEFAULT");
```

## Puntos Clave para Recordar

1. **Memoria**: Stack (variables locales) vs Heap (objetos)
2. **GC**: Conocer tipos y cuándo se ejecuta
3. **Colecciones**: Complejidad temporal de cada implementación
4. **Concurrencia**: Sincronización y herramientas modernas
5. **Streams**: Operaciones lazy vs eager
6. **Excepciones**: Checked vs unchecked
7. **Inmutabilidad**: Principio fundamental para thread-safety
