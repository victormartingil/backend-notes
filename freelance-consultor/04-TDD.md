# 🔺 Pirámide de Tests y Estrategia de Testing

Una buena estrategia de testing combina diferentes tipos de pruebas con distintos niveles de granularidad. La **pirámide de tests** es un modelo que guía cómo estructurar los tests para obtener confianza, velocidad y mantenibilidad.

---

## 🔺 Estructura de la pirámide de testing

```
        ▲
        |  E2E / UI Tests (lentos, frágiles, pocos)
        |-------------------------------------------
        |  Tests de integración (Kafka, DB, REST...)
        |-------------------------------------------
        |  Tests unitarios (rápidos, muchos, estables)
        ▼
```

---

## ✅ Objetivo por nivel

### 🔹 Unitarios

* Prueban lógica de negocio aislada (sin Spring, sin base de datos)
* Muy rápidos y confiables
* Base más amplia (mayor cantidad)

```kotlin
@Test
fun `should apply 10% discount`() {
  val price = discountCalculator.applyDiscount(100.0)
  assertThat(price).isEqualTo(90.0)
}
```

---

### 🔹 Integración

* Verifican colaboración entre capas (servicios + repositorios, eventos Kafka, etc.)
* Pueden usar Testcontainers, MockMvc, Wiremock, etc.

```kotlin
@Test
fun `should persist order and publish event`() {
  // Save order → verify DB and Kafka event
}
```

---

### 🔹 End-to-End (E2E)

* Validan flujos completos como usuario (REST desde frontend o API pública)
* Son lentos y frágiles, debe haber pocos y bien definidos

```bash
curl -X POST http://localhost:8080/orders -d '{...}'
```

---

## 🧠 Reglas prácticas para aplicar

* 70% tests unitarios, 20% de integración, 10% E2E (orientativo)
* Testear lo esencial, no todo: foco en reglas de negocio, edge cases y contratos públicos
* Mockea solo fuera del dominio (infraestructura, APIs externas)
* El dominio **nunca debe necesitar mocks**
* Usa datos de prueba realistas (fixtures, builders, factory methods)

---

## ✅ Qué se espera de ti como consultor

* Definir una estrategia clara de testing para cada microservicio
* Justificar por qué usar unitarios vs integración vs E2E
* Promover herramientas como: `testcontainers`, `wiremock`, `rest-assured`, `mockk`, `assertk`, `springmockk`
* Auditar la cobertura y calidad de los tests
* Formar a los equipos para mantener una pirámide sana y sostenible

---

## 🧪 Ejercicio para practicar

1. Escribe tests unitarios para el dominio `Order` (total, añadir línea, remover)
2. Escribe tests de integración con una base de datos embebida (Testcontainers)
3. Añade un test E2E para crear una orden desde la API REST
4. Revisa qué aporta cada test y cuál sería redundante

---

## 📚 Recursos

* Martin Fowler – [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
* Thoughtworks – Testing Strategies
* Kotlin + Spring Boot test repo: [rest-assured-kotlin-example](https://github.com/kittinunf/rest-assured-kotlin)

---
