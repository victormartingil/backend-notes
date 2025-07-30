# 🧬 Kotlin para Backend

Kotlin es un lenguaje moderno, expresivo y seguro, ideal para construir backends robustos y mantenibles con Spring Boot. Aprovecha su sintaxis concisa y sus potentes features para diseñar servicios limpios, testables y eficientes.

---

## 🎯 Objetivo de su uso en backend

* Reducir boilerplate frente a Java
* Escribir código más expresivo, seguro y funcional
* Promover patrones modernos: DSLs, sealed classes, extensión de funciones, etc.

---

## 🔧 Características clave para backend

### 🔹 Data classes

* Ideal para DTOs, entidades, requests/responses

```kotlin
data class Customer(val id: UUID, val name: String)
```

### 🔹 Sealed classes (para modelar respuestas o errores)

```kotlin
sealed class PaymentResult {
  object Success : PaymentResult()
  data class Failure(val reason: String) : PaymentResult()
}
```

### 🔹 Extension functions

* Permite enriquecer clases existentes sin herencia

```kotlin
fun String.toSlug(): String = this.lowercase().replace(" ", "-")
```

### 🔹 Null safety + `?.`, `?:`, `let`, `also`, `run`, `takeIf`

```kotlin
val email = user.email ?: "default@example.com"
val domain = user.email?.let { extractDomain(it) }
```

### 🔹 Smart casting + type inference

```kotlin
fun handle(result: PaymentResult) = when (result) {
  is PaymentResult.Success -> println("ok")
  is PaymentResult.Failure -> println(result.reason)
}
```

---

## 🧠 Buenas prácticas Kotlin en backend

* Evita usar `!!`, diseña el modelo para evitar nulos donde no tocan
* Usa `val` siempre que sea posible (inmutabilidad)
* Prefiere funciones puras, sin efectos colaterales
* Aprovecha named arguments y default values en funciones
* Usa `require`, `check`, `error` para validaciones internas

---

## ✅ Qué se espera de ti como consultor

* Escribir código idiomático y expresivo en Kotlin
* Corregir anti-patrones Java-style (getters/setters, new, util classes…)
* Enseñar a migrar servicios de Java a Kotlin de forma progresiva
* Aplicar Kotlin DSLs para config, builders o validaciones
* Usar Kotlin para definir modelos de dominio limpios y expresivos

---

## 🧪 Ejercicio para practicar

1. Define una sealed class `SubscriptionStatus` con estados `Active`, `Expired`, `Trial(daysLeft)`
2. Modela un `data class` `Subscription` con id, status y dates
3. Crea una función de extensión para saber si puede renovarse (`fun Subscription.canRenew(): Boolean`)
4. Añade tests unitarios para cada caso

---

## 📚 Recursos

* Documentación oficial: [kotlinlang.org](https://kotlinlang.org/docs/home.html)
* Artículo: *Effective Kotlin – Best Practices*
* Repositorio: [Kotlin backend examples](https://github.com/JetBrains/kotlin-examples/tree/master/jvm/backend)

---
