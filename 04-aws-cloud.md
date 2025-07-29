# AWS Cloud - Guía de Entrevista Senior Backend

## 1. Servicios Core de AWS

### EC2 (Elastic Compute Cloud)
- **Instancias virtuales**: Servidores escalables en la nube
- **Tipos de instancia**: t3.micro, m5.large, c5.xlarge, etc.
- **AMI (Amazon Machine Images)**: Templates de sistemas operativos
- **Security Groups**: Firewalls virtuales
- **Key Pairs**: Autenticación SSH

```bash
# Lanzar instancia EC2
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --security-group-ids sg-903004f8
```

### S3 (Simple Storage Service)
- **Almacenamiento de objetos**: Archivos estáticos, backups, data lakes
- **Buckets**: Contenedores para objetos
- **Storage Classes**: Standard, IA, Glacier, Deep Archive
- **Versionado**: Múltiples versiones del mismo objeto

```java
// Spring Boot + S3
@Service
public class S3Service {
    
    private final AmazonS3 s3Client;
    
    public String uploadFile(String bucketName, String key, MultipartFile file) {
        try {
            ObjectMetadata metadata = new ObjectMetadata();
            metadata.setContentLength(file.getSize());
            metadata.setContentType(file.getContentType());
            
            s3Client.putObject(bucketName, key, file.getInputStream(), metadata);
            return s3Client.getUrl(bucketName, key).toString();
        } catch (Exception e) {
            throw new RuntimeException("Error uploading file to S3", e);
        }
    }
    
    public String generatePresignedUrl(String bucketName, String key, int expirationMinutes) {
        Date expiration = Date.from(Instant.now().plusSeconds(expirationMinutes * 60));
        GeneratePresignedUrlRequest request = new GeneratePresignedUrlRequest(bucketName, key)
            .withMethod(HttpMethod.GET)
            .withExpiration(expiration);
        
        return s3Client.generatePresignedUrl(request).toString();
    }
}
```

### RDS (Relational Database Service)
- **Bases de datos gestionadas**: MySQL, PostgreSQL, Oracle, SQL Server
- **Multi-AZ**: Alta disponibilidad con failover automático
- **Read Replicas**: Réplicas de lectura para escalabilidad
- **Backup automático**: Point-in-time recovery

```yaml
# application.yml para RDS
spring:
  datasource:
    url: jdbc:postgresql://mydb.123456789012.us-east-1.rds.amazonaws.com:5432/myapp
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: none
```

## 2. Servicios de Contenedores

### ECS (Elastic Container Service)
- **Orquestación de contenedores**: Gestión de Docker containers
- **Task Definitions**: Blueprints para contenedores
- **Services**: Mantienen número deseado de tasks ejecutándose
- **Clusters**: Grupo de instancias EC2 o Fargate

```json
// Task Definition
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "my-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "SPRING_PROFILES_ACTIVE",
          "value": "prod"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### EKS (Elastic Kubernetes Service)
- **Kubernetes gestionado**: Control plane gestionado por AWS
- **Worker Nodes**: EC2 instances o Fargate
- **ALB Controller**: Application Load Balancer integration
- **IAM Roles**: Service accounts con roles IAM

```yaml
# Deployment Kubernetes en EKS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-service-account
      containers:
      - name: my-app
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## 3. Servicios Serverless

### Lambda
- **Functions as a Service**: Código sin gestión de servidores
- **Event-driven**: Responde a eventos de otros servicios AWS
- **Auto-scaling**: Escala automáticamente según demanda
- **Pay per use**: Solo pagas por tiempo de ejecución

```java
// Lambda function con Spring Cloud Function
@Component
public class UserHandler implements Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    @Autowired
    private UserService userService;
    
    @Override
    public APIGatewayProxyResponseEvent apply(APIGatewayProxyRequestEvent request) {
        try {
            String method = request.getHttpMethod();
            String path = request.getPath();
            
            if ("GET".equals(method) && path.startsWith("/users/")) {
                String userId = path.substring("/users/".length());
                User user = userService.findById(Long.valueOf(userId));
                
                return APIGatewayProxyResponseEvent.builder()
                    .withStatusCode(200)
                    .withBody(objectMapper.writeValueAsString(user))
                    .withHeaders(Map.of("Content-Type", "application/json"))
                    .build();
            }
            
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(404)
                .withBody("{\"error\":\"Not found\"}")
                .build();
                
        } catch (Exception e) {
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(500)
                .withBody("{\"error\":\"Internal server error\"}")
                .build();
        }
    }
}
```

### API Gateway
- **API Management**: Proxy para servicios backend
- **Authentication**: Integración con Cognito, IAM
- **Rate limiting**: Control de tráfico
- **Caching**: Cache de respuestas

```yaml
# Serverless Framework
service: my-api

provider:
  name: aws
  runtime: java11
  region: us-east-1

functions:
  getUser:
    handler: com.example.UserHandler
    events:
      - http:
          path: users/{id}
          method: get
          cors: true
          authorizer:
            name: auth
            arn: arn:aws:cognito-idp:us-east-1:123456789012:userpool/us-east-1_ABC123
```

## 4. Bases de Datos y Storage

### DynamoDB
- **NoSQL**: Base de datos de documentos y clave-valor
- **Serverless**: Auto-scaling y fully managed
- **Partition Key + Sort Key**: Esquema de claves
- **Global Secondary Indexes**: Consultas flexibles

```java
// DynamoDB con Spring Data
@DynamoDBTable(tableName = "Users")
public class User {
    
    @DynamoDBHashKey
    private String userId;
    
    @DynamoDBRangeKey
    private String email;
    
    @DynamoDBAttribute
    private String name;
    
    @DynamoDBAttribute
    private Integer age;
    
    @DynamoDBIndexHashKey(globalSecondaryIndexName = "email-index")
    public String getEmail() {
        return email;
    }
    
    // Getters y setters
}

@Repository
public interface UserRepository extends CrudRepository<User, String> {
    
    List<User> findByEmail(String email);
    
    @Query("SELECT * FROM Users WHERE #age > :age")
    List<User> findByAgeGreaterThan(@Param("age") Integer age);
}
```

### ElastiCache
- **In-memory caching**: Redis y Memcached
- **Session storage**: Almacenamiento de sesiones distribuido
- **Database caching**: Cache para reducir carga en BD

```java
// Spring Boot + Redis
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("my-cluster.abc123.cache.amazonaws.com", 6379)
        );
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public User save(User user) {
        return userRepository.save(user);
    }
}
```

## 5. Messaging y Event-Driven

### SQS (Simple Queue Service)
- **Message queues**: Desacoplamiento entre servicios
- **FIFO queues**: Orden garantizado
- **Dead Letter Queues**: Manejo de mensajes fallidos
- **Visibility timeout**: Tiempo para procesar mensaje

```java
// Spring Cloud AWS SQS
@Component
public class MessageProducer {
    
    @Autowired
    private QueueMessagingTemplate queueMessagingTemplate;
    
    public void sendMessage(UserEvent event) {
        queueMessagingTemplate.send("user-events-queue", 
            MessageBuilder.withPayload(event)
                .setHeader("eventType", event.getType())
                .build());
    }
}

@SqsListener("user-events-queue")
@Component
public class MessageConsumer {
    
    @Autowired
    private UserService userService;
    
    @SqsListener("user-events-queue")
    public void handleUserEvent(UserEvent event) {
        try {
            switch (event.getType()) {
                case "USER_CREATED":
                    userService.processNewUser(event.getUserId());
                    break;
                case "USER_UPDATED":
                    userService.processUserUpdate(event.getUserId());
                    break;
                default:
                    log.warn("Unknown event type: {}", event.getType());
            }
        } catch (Exception e) {
            log.error("Error processing user event", e);
            throw e; // Reintenta o envía a DLQ
        }
    }
}
```

## Puntos Clave para Recordar

1. **Compute**: EC2 (control total), ECS (containers), Lambda (serverless)
2. **Storage**: S3 (objetos), EBS (bloques), EFS (archivos)
3. **Database**: RDS (relacional), DynamoDB (NoSQL), ElastiCache (cache)
4. **Networking**: VPC (aislamiento), ALB (balanceador), CloudFront (CDN)
5. **Security**: IAM (identidades), Security Groups (firewall), Secrets Manager
6. **Monitoring**: CloudWatch (logs/métricas), X-Ray (tracing)
7. **Messaging**: SQS (queues), SNS (pub/sub), EventBridge (events)
8. **DevOps**: CodePipeline (CI/CD), CloudFormation (IaC)
