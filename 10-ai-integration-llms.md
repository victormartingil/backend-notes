# AI Integration & LLMs - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### Large Language Models (LLMs)
- **Definición**: Modelos de IA entrenados en grandes datasets de texto
- **Capacidades**: Generación de texto, comprensión, razonamiento, código
- **Modelos principales**: GPT-4, Claude, LLaMA, Gemini
- **API vs Self-hosted**: Trade-offs de costo, latencia, privacidad

### Tokens y Tokenización
```java
public class TokenCalculator {
    
    // Aproximación: 1 token ≈ 4 caracteres en inglés
    public int estimateTokens(String text) {
        return Math.round(text.length() / 4.0f);
    }
    
    public double calculateCost(int inputTokens, int outputTokens) {
        double inputCostPer1K = 0.03;   // $0.03 per 1K input tokens
        double outputCostPer1K = 0.06;  // $0.06 per 1K output tokens
        
        return (inputTokens / 1000.0 * inputCostPer1K) + 
               (outputTokens / 1000.0 * outputCostPer1K);
    }
}
```

## 2. OpenAI API Integration

### Configuración y Client
```java
@Configuration
public class OpenAIConfig {
    
    @Value("${openai.api.key}")
    private String apiKey;
    
    @Bean
    public OpenAIClient openAIClient() {
        return OpenAIClient.builder()
            .apiKey(apiKey)
            .timeout(Duration.ofMinutes(2))
            .build();
    }
}

@Service
public class OpenAIService {
    
    private final OpenAIClient openAIClient;
    
    public String generateCompletion(String prompt, String model) {
        try {
            ChatCompletionRequest request = ChatCompletionRequest.builder()
                .model(model)
                .messages(List.of(
                    ChatMessage.builder()
                        .role("user")
                        .content(prompt)
                        .build()
                ))
                .maxTokens(1000)
                .temperature(0.7)
                .build();
            
            ChatCompletionResponse response = openAIClient.createChatCompletion(request);
            return response.getChoices().get(0).getMessage().getContent();
            
        } catch (Exception e) {
            throw new AIServiceException("Failed to generate completion", e);
        }
    }
}
```

## 3. RAG (Retrieval-Augmented Generation)

### Vector Embeddings
```java
@Service
public class EmbeddingService {
    
    public List<Double> generateEmbedding(String text) {
        EmbeddingRequest request = EmbeddingRequest.builder()
            .model("text-embedding-ada-002")
            .input(List.of(text))
            .build();
        
        EmbeddingResponse response = openAIClient.createEmbedding(request);
        return response.getData().get(0).getEmbedding();
    }
    
    public double cosineSimilarity(List<Double> vectorA, List<Double> vectorB) {
        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;
        
        for (int i = 0; i < vectorA.size(); i++) {
            dotProduct += vectorA.get(i) * vectorB.get(i);
            normA += Math.pow(vectorA.get(i), 2);
            normB += Math.pow(vectorB.get(i), 2);
        }
        
        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### RAG Service
```java
@Service
public class RAGService {
    
    private final EmbeddingService embeddingService;
    private final DocumentChunkRepository chunkRepository;
    private final OpenAIService openAIService;
    
    public String answerQuestion(String question) {
        // 1. Generate embedding for question
        List<Double> questionEmbedding = embeddingService.generateEmbedding(question);
        
        // 2. Find relevant documents
        List<DocumentChunk> relevantChunks = findRelevantChunks(questionEmbedding);
        
        // 3. Create context
        String context = relevantChunks.stream()
            .map(DocumentChunk::getContent)
            .collect(Collectors.joining("\n\n"));
        
        // 4. Generate answer with context
        String prompt = """
            Answer based on the context provided.
            
            Context: %s
            Question: %s
            Answer:
            """.formatted(context, question);
        
        return openAIService.generateCompletion(prompt, "gpt-4");
    }
}
```

## 4. Function Calling

```java
// Function Definitions
public class OpenAIFunctions {
    
    public static ChatCompletionFunction getDatabaseQueryFunction() {
        return ChatCompletionFunction.builder()
            .name("query_database")
            .description("Execute a SQL query")
            .parameters(JsonSchema.builder()
                .type("object")
                .properties(Map.of(
                    "query", JsonSchema.builder()
                        .type("string")
                        .description("SQL query to execute")
                        .build()
                ))
                .required(List.of("query"))
                .build())
            .build();
    }
}

// Function Execution
@Service
public class FunctionExecutionService {
    
    public String executeFunction(String functionName, String arguments) {
        try {
            JsonNode argsNode = objectMapper.readTree(arguments);
            
            return switch (functionName) {
                case "query_database" -> {
                    String query = argsNode.get("query").asText();
                    
                    if (!isQuerySafe(query)) {
                        throw new SecurityException("Query not allowed");
                    }
                    
                    List<Map<String, Object>> results = databaseService.executeQuery(query);
                    yield objectMapper.writeValueAsString(results);
                }
                default -> throw new IllegalArgumentException("Unknown function");
            };
        } catch (Exception e) {
            return "{\"error\": \"Function execution failed\"}";
        }
    }
    
    private boolean isQuerySafe(String query) {
        String upperQuery = query.toUpperCase();
        return upperQuery.startsWith("SELECT") && 
               !upperQuery.contains("DROP");
    }
}
```

## 5. Security y Best Practices

### Rate Limiting
```java
@Component
public class AIRateLimiter {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean isAllowed(String userId, int maxRequests, Duration window) {
        String key = "ai_rate_limit:" + userId;
        String currentCount = redisTemplate.opsForValue().get(key);
        
        if (currentCount == null) {
            redisTemplate.opsForValue().set(key, "1", window);
            return true;
        }
        
        int count = Integer.parseInt(currentCount);
        if (count >= maxRequests) {
            return false;
        }
        
        redisTemplate.opsForValue().increment(key);
        return true;
    }
}
```

### Prompt Security
```java
@Service
public class PromptSecurityService {
    
    private final List<String> dangerousPatterns = List.of(
        "ignore previous instructions",
        "forget everything above",
        "jailbreak"
    );
    
    public boolean isPromptSafe(String prompt) {
        String lowerPrompt = prompt.toLowerCase();
        
        for (String pattern : dangerousPatterns) {
            if (lowerPrompt.contains(pattern)) {
                return false;
            }
        }
        
        return true;
    }
    
    public String sanitizePrompt(String prompt) {
        return prompt
            .replaceAll("[<>]", "")
            .trim();
    }
}
```

## 6. Preguntas Típicas de Entrevista

### ¿Qué es RAG y cuándo usarlo?
- **RAG**: Retrieval-Augmented Generation
- **Casos de uso**: Knowledge bases, documentación empresarial
- **Ventajas**: Información actualizada, menos alucinaciones
- **Implementación**: Embeddings + Vector DB + LLM

### ¿Cómo manejar el context window?
- **Limitaciones**: GPT-4 tiene ~8k-32k tokens
- **Estrategias**: Chunking, summarization
- **Token management**: Optimizar para reducir costos

### ¿Qué son los embeddings?
- **Definición**: Representación vectorial de texto
- **Propiedades**: Textos similares → embeddings cercanos
- **Uso**: Búsqueda semántica, clustering

## 7. Mejores Prácticas

### Cost Optimization
```java
@Service
public class CostOptimizationService {
    
    public String optimizePrompt(String prompt) {
        return prompt
            .replaceAll("\\s+", " ")
            .replaceAll("(?i)please|thank you", "")
            .trim();
    }
    
    public boolean shouldUseCache(String prompt) {
        return !prompt.toLowerCase().contains("current") &&
               !prompt.toLowerCase().contains("today");
    }
}
```

### Error Handling
```java
@Component
public class AIErrorHandler {
    
    @Retryable(maxAttempts = 3)
    public String generateWithRetry(String prompt) {
        return openAIService.generateCompletion(prompt, "gpt-4");
    }
    
    @Recover
    public String recover(Exception ex) {
        return "I'm sorry, service temporarily unavailable.";
    }
}
```

## Puntos Clave para Recordar

1. **Token Management**: Optimizar para reducir costos
2. **Rate Limiting**: Controlar uso y proteger APIs
3. **Security**: Validar inputs, prevenir injection
4. **RAG**: Combinar búsqueda con generación
5. **Function Calling**: Extender capacidades del LLM
6. **Caching**: Reutilizar respuestas similares
7. **Error Handling**: Retry logic y fallbacks
8. **Monitoring**: Métricas de uso y calidad
