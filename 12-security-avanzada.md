# Security Avanzada - Guía de Entrevista Senior Backend

## 1. OAuth 2.0 & OpenID Connect

### Conceptos Fundamentales
- **OAuth 2.0**: Framework de autorización
- **OpenID Connect**: Capa de identidad sobre OAuth 2.0
- **JWT**: JSON Web Tokens para transportar información
- **Scopes**: Permisos granulares

### Implementación con Spring Security
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/users/**").hasAuthority("SCOPE_read")
                .requestMatchers(HttpMethod.POST, "/api/users/**").hasAuthority("SCOPE_write")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(jwtDecoder()))
            )
            .build();
    }
}
```

## 2. API Security

### Rate Limiting
```java
@Component
public class RedisRateLimiter {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "rate_limit:" + key;
        
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        
        // Remove old entries
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        
        // Count current requests
        Long currentRequests = redisTemplate.opsForZSet().count(redisKey, windowStart, now);
        
        if (currentRequests >= maxRequests) {
            return false;
        }
        
        // Add current request
        redisTemplate.opsForZSet().add(redisKey, UUID.randomUUID().toString(), now);
        redisTemplate.expire(redisKey, window);
        
        return true;
    }
}
```

### Input Validation
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NoSqlInjectionValidator.class)
public @interface NoSqlInjection {
    String message() default "Input contains potential SQL injection";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class NoSqlInjectionValidator implements ConstraintValidator<NoSqlInjection, String> {
    
    private static final Pattern SQL_INJECTION_PATTERN = Pattern.compile(
        "(?i)(union|select|insert|update|delete|drop|create|alter|exec|script)"
    );
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;
        }
        return !SQL_INJECTION_PATTERN.matcher(value).find();
    }
}
```

## 3. Secrets Management

```java
@Service
public class SecretsService {
    
    private final AWSSecretsManager secretsManager;
    private final Cache<String, String> secretCache;
    
    public String getSecret(String secretName) {
        return secretCache.get(secretName, this::fetchSecretFromAWS);
    }
    
    private String fetchSecretFromAWS(String secretName) {
        try {
            GetSecretValueRequest request = new GetSecretValueRequest()
                .withSecretId(secretName);
                
            GetSecretValueResult result = secretsManager.getSecretValue(request);
            return result.getSecretString();
            
        } catch (Exception e) {
            throw new SecurityException("Failed to fetch secret", e);
        }
    }
}
```

## 4. Encryption

```java
@Component
public class AESUtil {
    
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private final SecretKey secretKey;
    
    public String encrypt(String plaintext) throws Exception {
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);
        byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
        
        // Combine IV + ciphertext
        byte[] encryptedWithIv = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, encryptedWithIv, 0, iv.length);
        System.arraycopy(ciphertext, 0, encryptedWithIv, iv.length, ciphertext.length);
        
        return Base64.getEncoder().encodeToString(encryptedWithIv);
    }
}
```

## 5. Secure Coding Practices

### SQL Injection Prevention
```java
@Repository
public class UserRepository {
    
    // ✅ Correct: Using parameterized queries
    public List<User> findUsersByStatus(UserStatus status) {
        String sql = "SELECT * FROM users WHERE status = ?";
        return jdbcTemplate.query(sql, 
            new Object[]{status.name()}, 
            new UserRowMapper());
    }
    
    // ❌ Wrong: String concatenation (vulnerable)
    public List<User> findUsersByNameWrong(String name) {
        String sql = "SELECT * FROM users WHERE name = '" + name + "'";
        return jdbcTemplate.query(sql, new UserRowMapper());
    }
    
    // ✅ Correct: Input validation + parameterized query
    public List<User> searchUsers(String searchTerm, String sortColumn) {
        if (!isValidSortColumn(sortColumn)) {
            throw new IllegalArgumentException("Invalid sort column");
        }
        
        String sql = "SELECT * FROM users WHERE name LIKE ? ORDER BY " + sortColumn;
        return jdbcTemplate.query(sql, 
            new Object[]{"%" + searchTerm + "%"}, 
            new UserRowMapper());
    }
    
    private boolean isValidSortColumn(String column) {
        Set<String> allowedColumns = Set.of("name", "email", "created_at");
        return allowedColumns.contains(column.toLowerCase());
    }
}
```

### XSS Prevention
```java
@RestController
public class ContentController {
    
    @PostMapping("/api/content")
    public ResponseEntity<ContentResponse> createContent(
            @RequestBody @Valid CreateContentRequest request) {
        
        // Sanitize HTML content
        String sanitizedContent = inputSanitizer.sanitizeHtml(request.getContent());
        
        Content content = new Content(
            request.getTitle(),
            sanitizedContent,
            getCurrentUserId()
        );
        
        Content savedContent = contentService.save(content);
        return ResponseEntity.ok(new ContentResponse(savedContent));
    }
}

// Security Headers Filter
@Component
public class SecurityHeadersFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        httpResponse.setHeader("Content-Security-Policy", 
            "default-src 'self'; script-src 'self'; frame-ancestors 'none';");
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("X-Frame-Options", "DENY");
        httpResponse.setHeader("X-XSS-Protection", "1; mode=block");
        
        chain.doFilter(request, response);
    }
}
```

## 6. Security Testing

```java
@SpringBootTest
class SecurityTest {
    
    @Test
    void shouldPreventSQLInjection() throws Exception {
        String maliciousInput = "'; DROP TABLE users; --";
        
        mockMvc.perform(get("/api/users/search")
                .param("name", maliciousInput))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data").isEmpty());
        
        // Verify table still exists
        long userCount = userRepository.count();
        assertThat(userCount).isGreaterThan(0);
    }
    
    @Test
    void shouldPreventXSSAttacks() throws Exception {
        String xssPayload = "<script>alert('XSS')</script>";
        
        CreateContentRequest request = new CreateContentRequest(
            "Test Title",
            xssPayload
        );
        
        mockMvc.perform(post("/api/content")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk());
        
        // Verify script tags are removed
        Content savedContent = contentRepository.findByTitle("Test Title");
        assertThat(savedContent.getHtmlContent()).doesNotContain("<script>");
    }
    
    @Test
    void shouldEnforceRateLimit() throws Exception {
        String endpoint = "/api/auth/login";
        
        // Make requests up to the limit
        for (int i = 0; i < 5; i++) {
            mockMvc.perform(post(endpoint)
                    .content("{\"username\":\"test\",\"password\":\"wrong\"}"))
                    .andExpect(status().isUnauthorized());
        }
        
        // Next request should be rate limited
        mockMvc.perform(post(endpoint)
                .content("{\"username\":\"test\",\"password\":\"wrong\"}"))
                .andExpect(status().isTooManyRequests())
                .andExpect(header().exists("Retry-After"));
    }
}
```

## 7. Preguntas Típicas de Entrevista

### ¿Qué es OAuth 2.0 y cuáles son sus flows?
- **Authorization Code**: Para aplicaciones web con backend
- **PKCE**: Para SPAs y aplicaciones móviles  
- **Client Credentials**: Para comunicación service-to-service
- **Resource Owner Password**: Legacy, no recomendado

### ¿Cómo prevenir ataques de SQL Injection?
- **Prepared Statements**: Usar parámetros en lugar de concatenación
- **Input Validation**: Validar y sanitizar entradas
- **Least Privilege**: Permisos mínimos para BD
- **Stored Procedures**: Cuando sea apropiado

### ¿Qué es CORS y cómo configurarlo de forma segura?
- **Same-Origin Policy**: Restricción del navegador
- **CORS Headers**: Access-Control-Allow-* headers
- **Security**: Nunca usar wildcard (*) con credentials
- **Preflight**: Requests OPTIONS para operaciones complejas

### ¿Cómo manejar autenticación en microservicios?
- **JWT**: Tokens stateless entre servicios
- **API Gateway**: Autenticación centralizada
- **Service Mesh**: Mutual TLS (mTLS)
- **Token Relay**: Propagar tokens entre servicios

## 8. Mejores Prácticas

### Security Headers
```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentTypeOptions(ContentTypeOptionsConfig::and)
                .frameOptions(FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(jwtDecoder()))
            )
            .build();
    }
}
```

### Audit Logging
```java
@Component
public class SecurityAuditLogger {
    
    private final Logger auditLogger = LoggerFactory.getLogger("SECURITY_AUDIT");
    
    public void logAuthenticationSuccess(String username, String ipAddress) {
        auditLogger.info("Authentication successful - User: {}, IP: {}",
            username, ipAddress);
    }
    
    public void logAuthenticationFailure(String username, String ipAddress, String reason) {
        auditLogger.warn("Authentication failed - User: {}, IP: {}, Reason: {}",
            username, ipAddress, reason);
    }
    
    public void logSecurityViolation(String type, String details, String ipAddress) {
        auditLogger.error("Security violation - Type: {}, Details: {}, IP: {}",
            type, details, ipAddress);
    }
}
```

## Puntos Clave para Recordar

1. **Defense in Depth**: Múltiples capas de seguridad
2. **Least Privilege**: Mínimos permisos necesarios
3. **Input Validation**: Validar y sanitizar todas las entradas
4. **Secure by Default**: Configuraciones seguras por defecto
5. **Encryption**: Datos en tránsito y en reposo
6. **Audit Logging**: Registrar eventos de seguridad
7. **Regular Updates**: Mantener dependencias actualizadas
8. **Security Testing**: Integrar en CI/CD
9. **Incident Response**: Plan para manejar brechas
10. **Security Training**: Educación continua del equipo
