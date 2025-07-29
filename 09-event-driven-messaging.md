# Event-Driven Architecture & Messaging - Guía de Entrevista Senior Backend

## 1. Conceptos Fundamentales

### Event-Driven Architecture (EDA)
- **Definición**: Arquitectura donde los componentes se comunican a través de eventos
- **Ventajas**: Desacoplamiento, escalabilidad, resistencia a fallos
- **Patrones**: Pub/Sub, Event Sourcing, CQRS, Saga

### Tipos de Eventos
```java
// Domain Event
public class UserRegisteredEvent {
    private final UserId userId;
    private final Email email;
    private final LocalDateTime occurredAt;
    
    public UserRegisteredEvent(UserId userId, Email email) {
        this.userId = userId;
        this.email = email;
        this.occurredAt = LocalDateTime.now();
    }
    
    // Getters
}

// Integration Event
public class OrderShippedEvent {
    private final String orderId;
    private final String customerId;
    private final String trackingNumber;
    private final LocalDateTime shippedAt;
    
    // Serializable para comunicación entre servicios
}

// Command vs Event
public class CreateUserCommand {
    // Intención de hacer algo
    private final String name;
    private final String email;
}

public class UserCreatedEvent {
    // Algo que ya pasó
    private final UserId userId;
    private final LocalDateTime createdAt;
}
```

## 2. Apache Kafka

### Conceptos Clave
- **Topics**: Canales de comunicación
- **Partitions**: Subdivisiones para escalabilidad
- **Producers**: Publican mensajes
- **Consumers**: Consumen mensajes
- **Consumer Groups**: Distribución de carga

### Configuración en Spring Boot
```java
// Producer Configuration
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        configProps.put(ProducerConfig.IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

// Consumer Configuration
@Configuration
@EnableKafka
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "user-service-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }
}
```

### Producer y Consumer
```java
// Event Publisher
@Service
public class EventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public EventPublisher(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void publishUserRegistered(UserRegisteredEvent event) {
        try {
            ListenableFuture<SendResult<String, Object>> future = 
                kafkaTemplate.send("user-events", event.getUserId().toString(), event);
                
            future.addCallback(
                result -> log.info("Event published successfully: {}", event),
                failure -> log.error("Failed to publish event: {}", event, failure)
            );
        } catch (Exception e) {
            log.error("Error publishing event", e);
            throw new EventPublishingException("Failed to publish user registered event", e);
        }
    }
}

// Event Consumer
@Component
public class UserEventConsumer {
    
    @KafkaListener(topics = "user-events", groupId = "notification-service-group")
    public void handleUserRegistered(
            @Payload UserRegisteredEvent event,
            @Header Map<String, Object> headers,
            Acknowledgment acknowledgment) {
        
        try {
            log.info("Processing user registered event: {}", event);
            
            // Process event
            emailService.sendWelcomeEmail(event.getEmail());
            
            // Manual acknowledgment
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Error processing user registered event", e);
            // No acknowledge - message will be retried
        }
    }
    
    @KafkaListener(topics = "user-events", groupId = "analytics-service-group")
    public void handleUserRegisteredForAnalytics(UserRegisteredEvent event) {
        // Different consumer group for analytics
        analyticsService.trackUserRegistration(event);
    }
}
```

### Error Handling y Dead Letter Queue
```java
@Component
public class KafkaErrorHandler {
    
    @Bean
    public ConsumerAwareListenerErrorHandler kafkaErrorHandler() {
        return (message, exception, consumer) -> {
            log.error("Error processing Kafka message: {}", message.getPayload(), exception);
            
            // Send to DLQ
            kafkaTemplate.send("user-events-dlq", message.getPayload());
            
            return null;
        };
    }
    
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 1000, multiplier = 2),
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    @KafkaListener(topics = "user-events")
    public void handleWithRetry(UserRegisteredEvent event) {
        // Processing logic with automatic retry
        processEvent(event);
    }
}
```

## 3. Event Sourcing

### Conceptos Fundamentales
- **Event Store**: Almacena secuencia de eventos
- **Snapshots**: Estados consolidados para performance
- **Projections**: Vistas derivadas de eventos
- **Replay**: Reconstruir estado desde eventos

### Implementación Básica
```java
// Event Store Interface
public interface EventStore {
    void saveEvents(String aggregateId, List<DomainEvent> events, int expectedVersion);
    List<DomainEvent> getEventsForAggregate(String aggregateId);
    List<DomainEvent> getEventsForAggregateFromVersion(String aggregateId, int version);
}

// Event Store Implementation
@Repository
public class JpaEventStore implements EventStore {
    
    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    
    @Override
    @Transactional
    public void saveEvents(String aggregateId, List<DomainEvent> events, int expectedVersion) {
        // Optimistic concurrency control
        int currentVersion = getCurrentVersion(aggregateId);
        if (currentVersion != expectedVersion) {
            throw new ConcurrencyException("Expected version " + expectedVersion + 
                " but current version is " + currentVersion);
        }
        
        for (int i = 0; i < events.size(); i++) {
            DomainEvent event = events.get(i);
            EventStoreEntry entry = new EventStoreEntry(
                aggregateId,
                event.getClass().getSimpleName(),
                objectMapper.writeValueAsString(event),
                expectedVersion + i + 1,
                LocalDateTime.now()
            );
            repository.save(entry);
        }
    }
    
    @Override
    public List<DomainEvent> getEventsForAggregate(String aggregateId) {
        return repository.findByAggregateIdOrderByVersion(aggregateId)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
}

// Aggregate Root con Event Sourcing
public abstract class EventSourcedAggregateRoot {
    private String id;
    private int version = 0;
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    protected void addEvent(DomainEvent event) {
        uncommittedEvents.add(event);
        apply(event);
    }
    
    public void loadFromHistory(List<DomainEvent> history) {
        for (DomainEvent event : history) {
            apply(event);
            version++;
        }
    }
    
    public List<DomainEvent> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }
    
    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
    
    protected abstract void apply(DomainEvent event);
    
    // Getters
    public int getVersion() { return version; }
    public String getId() { return id; }
}

// User Aggregate con Event Sourcing
public class User extends EventSourcedAggregateRoot {
    private UserId userId;
    private String name;
    private String email;
    private UserStatus status;
    
    // Constructor para nuevos usuarios
    public User(UserId userId, String name, String email) {
        UserCreatedEvent event = new UserCreatedEvent(userId, name, email);
        addEvent(event);
    }
    
    // Constructor para reconstruir desde eventos
    public User() {}
    
    public void updateEmail(String newEmail) {
        if (!newEmail.equals(this.email)) {
            EmailUpdatedEvent event = new EmailUpdatedEvent(userId, email, newEmail);
            addEvent(event);
        }
    }
    
    public void deactivate() {
        if (status == UserStatus.ACTIVE) {
            UserDeactivatedEvent event = new UserDeactivatedEvent(userId);
            addEvent(event);
        }
    }
    
    @Override
    protected void apply(DomainEvent event) {
        if (event instanceof UserCreatedEvent) {
            apply((UserCreatedEvent) event);
        } else if (event instanceof EmailUpdatedEvent) {
            apply((EmailUpdatedEvent) event);
        } else if (event instanceof UserDeactivatedEvent) {
            apply((UserDeactivatedEvent) event);
        }
    }
    
    private void apply(UserCreatedEvent event) {
        this.userId = event.getUserId();
        this.name = event.getName();
        this.email = event.getEmail();
        this.status = UserStatus.ACTIVE;
    }
    
    private void apply(EmailUpdatedEvent event) {
        this.email = event.getNewEmail();
    }
    
    private void apply(UserDeactivatedEvent event) {
        this.status = UserStatus.INACTIVE;
    }
}
```

### Projections y Read Models
```java
// Projection Handler
@Component
public class UserProjectionHandler {
    
    private final UserReadModelRepository readModelRepository;
    
    @EventHandler
    public void handle(UserCreatedEvent event) {
        UserReadModel readModel = new UserReadModel(
            event.getUserId().toString(),
            event.getName(),
            event.getEmail(),
            UserStatus.ACTIVE.name(),
            event.getOccurredAt()
        );
        
        readModelRepository.save(readModel);
    }
    
    @EventHandler
    public void handle(EmailUpdatedEvent event) {
        UserReadModel readModel = readModelRepository.findById(event.getUserId().toString())
            .orElseThrow(() -> new ReadModelNotFoundException("User read model not found"));
            
        readModel.setEmail(event.getNewEmail());
        readModel.setLastModified(event.getOccurredAt());
        
        readModelRepository.save(readModel);
    }
    
    @EventHandler
    public void handle(UserDeactivatedEvent event) {
        UserReadModel readModel = readModelRepository.findById(event.getUserId().toString())
            .orElseThrow(() -> new ReadModelNotFoundException("User read model not found"));
            
        readModel.setStatus(UserStatus.INACTIVE.name());
        readModel.setLastModified(event.getOccurredAt());
        
        readModelRepository.save(readModel);
    }
}
```

## 4. Saga Pattern

### Orchestration vs Choreography
```java
// Choreography - Eventos coordinan el flujo
@Component
public class OrderSagaChoreography {
    
    @EventHandler
    public void handle(OrderCreatedEvent event) {
        // Reserve inventory
        InventoryReservationCommand command = new InventoryReservationCommand(
            event.getOrderId(),
            event.getItems()
        );
        commandGateway.send(command);
    }
    
    @EventHandler
    public void handle(InventoryReservedEvent event) {
        // Process payment
        ProcessPaymentCommand command = new ProcessPaymentCommand(
            event.getOrderId(),
            event.getTotalAmount(),
            event.getPaymentDetails()
        );
        commandGateway.send(command);
    }
    
    @EventHandler
    public void handle(PaymentProcessedEvent event) {
        // Confirm order
        ConfirmOrderCommand command = new ConfirmOrderCommand(event.getOrderId());
        commandGateway.send(command);
    }
    
    @EventHandler
    public void handle(PaymentFailedEvent event) {
        // Compensate inventory reservation
        ReleaseInventoryCommand command = new ReleaseInventoryCommand(event.getOrderId());
        commandGateway.send(command);
    }
}

// Orchestration - Saga centralizada controla el flujo
@Saga
public class OrderSagaOrchestration {
    
    @Autowired
    private transient CommandGateway commandGateway;
    
    private String orderId;
    private boolean inventoryReserved = false;
    private boolean paymentProcessed = false;
    
    @SagaOrchestrationStart
    @SagaAssociationProperty("orderId")
    public void handle(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        
        // Step 1: Reserve inventory
        InventoryReservationCommand command = new InventoryReservationCommand(
            event.getOrderId(),
            event.getItems()
        );
        commandGateway.send(command);
    }
    
    @SagaAssociationProperty("orderId")
    public void handle(InventoryReservedEvent event) {
        this.inventoryReserved = true;
        
        // Step 2: Process payment
        ProcessPaymentCommand command = new ProcessPaymentCommand(
            event.getOrderId(),
            event.getTotalAmount(),
            event.getPaymentDetails()
        );
        commandGateway.send(command);
    }
    
    @SagaAssociationProperty("orderId")
    public void handle(PaymentProcessedEvent event) {
        this.paymentProcessed = true;
        
        // Step 3: Confirm order
        ConfirmOrderCommand command = new ConfirmOrderCommand(event.getOrderId());
        commandGateway.send(command);
    }
    
    @SagaAssociationProperty("orderId")
    public void handle(PaymentFailedEvent event) {
        // Compensate: Release inventory if it was reserved
        if (inventoryReserved) {
            ReleaseInventoryCommand command = new ReleaseInventoryCommand(event.getOrderId());
            commandGateway.send(command);
        }
        
        // Mark order as failed
        FailOrderCommand command = new FailOrderCommand(event.getOrderId(), "Payment failed");
        commandGateway.send(command);
    }
}
```

## 5. Message Queues (SQS, RabbitMQ)

### Amazon SQS
```java
// SQS Configuration
@Configuration
public class SqsConfig {
    
    @Bean
    public QueueMessagingTemplate queueMessagingTemplate() {
        return new QueueMessagingTemplate(amazonSQSAsync());
    }
    
    @Bean
    public AmazonSQSAsync amazonSQSAsync() {
        return AmazonSQSAsyncClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .build();
    }
}

// SQS Producer
@Service
public class SqsMessageProducer {
    
    private final QueueMessagingTemplate queueMessagingTemplate;
    
    public void sendMessage(Object message) {
        queueMessagingTemplate.send("user-events-queue", 
            MessageBuilder.withPayload(message)
                .setHeader("messageType", message.getClass().getSimpleName())
                .setHeader("timestamp", System.currentTimeMillis())
                .build());
    }
    
    public void sendMessageWithDelay(Object message, int delaySeconds) {
        queueMessagingTemplate.send("user-events-queue", 
            MessageBuilder.withPayload(message)
                .setHeader(SqsMessageHeaders.SQS_DELAY_HEADER, delaySeconds)
                .build());
    }
}

// SQS Consumer
@Component
public class SqsMessageConsumer {
    
    @SqsListener("user-events-queue")
    public void receiveMessage(
            UserRegisteredEvent event,
            @Header Map<String, String> headers) {
        
        try {
            log.info("Received message: {} with headers: {}", event, headers);
            processUserRegistration(event);
        } catch (Exception e) {
            log.error("Error processing message", e);
            throw e; // Message will go to DLQ after retries
        }
    }
    
    @SqsListener(value = "user-events-dlq", deletionPolicy = SqsMessageDeletionPolicy.NEVER)
    public void receiveMessageFromDlq(UserRegisteredEvent event) {
        log.error("Processing message from DLQ: {}", event);
        // Special handling for failed messages
        sendAlertToOperations(event);
    }
}
```

### RabbitMQ
```java
// RabbitMQ Configuration
@Configuration
@EnableRabbit
public class RabbitConfig {
    
    @Bean
    public TopicExchange userExchange() {
        return new TopicExchange("user.exchange");
    }
    
    @Bean
    public Queue userRegisteredQueue() {
        return QueueBuilder.durable("user.registered.queue")
            .withArgument("x-dead-letter-exchange", "user.dlx")
            .withArgument("x-dead-letter-routing-key", "user.registered.dlq")
            .build();
    }
    
    @Bean
    public Binding userRegisteredBinding() {
        return BindingBuilder
            .bind(userRegisteredQueue())
            .to(userExchange())
            .with("user.registered");
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(new Jackson2JsonMessageConverter());
        return template;
    }
}

// RabbitMQ Producer
@Service
public class RabbitMessageProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    public void publishUserRegistered(UserRegisteredEvent event) {
        rabbitTemplate.convertAndSend(
            "user.exchange",
            "user.registered",
            event,
            message -> {
                message.getMessageProperties().setMessageId(UUID.randomUUID().toString());
                message.getMessageProperties().setTimestamp(new Date());
                return message;
            }
        );
    }
}

// RabbitMQ Consumer
@Component
public class RabbitMessageConsumer {
    
    @RabbitListener(queues = "user.registered.queue")
    public void handleUserRegistered(
            UserRegisteredEvent event,
            @Header Map<String, Object> headers,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        
        try {
            log.info("Processing user registered event: {}", event);
            processUserRegistration(event);
            
            // Manual acknowledgment
            channel.basicAck(deliveryTag, false);
            
        } catch (Exception e) {
            log.error("Error processing user registered event", e);
            try {
                // Reject and requeue (or send to DLQ)
                channel.basicNack(deliveryTag, false, false);
            } catch (IOException ioException) {
                log.error("Error rejecting message", ioException);
            }
        }
    }
}
```

## 6. Preguntas Típicas de Entrevista

### ¿Cuál es la diferencia entre Event Sourcing y CQRS?
- **Event Sourcing**: Patrón de persistencia que almacena eventos en lugar de estado
- **CQRS**: Patrón que separa comandos (escritura) de consultas (lectura)
- **Relación**: Pueden usarse juntos pero son independientes
- **Beneficios**: Event Sourcing da auditabilidad, CQRS da escalabilidad

### ¿Cuándo usar Saga vs transacciones distribuidas?
- **Saga**: Para flujos de negocio largos, mejor UX, eventual consistency
- **2PC**: Para consistency fuerte, operaciones cortas, menos resiliente
- **Trade-offs**: Saga es más complejo pero más escalable

### ¿Diferencia entre At-least-once vs Exactly-once delivery?
- **At-least-once**: Mensaje puede llegar múltiples veces (idempotencia necesaria)
- **Exactly-once**: Mensaje llega exactamente una vez (más costoso)
- **Implementación**: Deduplicación, keys idempotentes

### ¿Qué es el patrón Outbox?
```java
@Transactional
public void createUserWithEvents(User user) {
    // Save user
    userRepository.save(user);
    
    // Save events in same transaction
    OutboxEvent outboxEvent = new OutboxEvent(
        user.getId(),
        "UserCreated",
        jsonSerialize(new UserCreatedEvent(user))
    );
    outboxRepository.save(outboxEvent);
    
    // Background process publishes events from outbox
}
```

## 7. Mejores Prácticas

### Idempotencia
```java
@Service
public class IdempotentMessageProcessor {
    
    private final MessageIdRepository messageIdRepository;
    
    public void processMessage(String messageId, UserRegisteredEvent event) {
        // Check if already processed
        if (messageIdRepository.existsById(messageId)) {
            log.info("Message {} already processed, skipping", messageId);
            return;
        }
        
        try {
            // Process message
            processUserRegistration(event);
            
            // Mark as processed
            messageIdRepository.save(new ProcessedMessageId(messageId, LocalDateTime.now()));
            
        } catch (Exception e) {
            log.error("Error processing message {}", messageId, e);
            throw e;
        }
    }
}
```

### Versionado de Eventos
```java
// Event versioning strategy
public abstract class DomainEvent {
    private final int version;
    private final LocalDateTime occurredAt;
    
    protected DomainEvent(int version) {
        this.version = version;
        this.occurredAt = LocalDateTime.now();
    }
}

public class UserCreatedEventV1 extends DomainEvent {
    private final String userId;
    private final String name;
    private final String email;
    
    public UserCreatedEventV1(String userId, String name, String email) {
        super(1);
        this.userId = userId;
        this.name = name;
        this.email = email;
    }
}

public class UserCreatedEventV2 extends DomainEvent {
    private final String userId;
    private final String firstName;
    private final String lastName;
    private final String email;
    private final String phoneNumber; // New field
    
    public UserCreatedEventV2(String userId, String firstName, String lastName, 
                             String email, String phoneNumber) {
        super(2);
        this.userId = userId;
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.phoneNumber = phoneNumber;
    }
}

// Event upcasting
@Component
public class EventUpcaster {
    
    public DomainEvent upcast(DomainEvent event) {
        if (event instanceof UserCreatedEventV1) {
            UserCreatedEventV1 v1 = (UserCreatedEventV1) event;
            String[] nameParts = v1.getName().split(" ", 2);
            return new UserCreatedEventV2(
                v1.getUserId(),
                nameParts[0],
                nameParts.length > 1 ? nameParts[1] : "",
                v1.getEmail(),
                null // No phone number in V1
            );
        }
        return event;
    }
}
```

## Puntos Clave para Recordar

1. **EDA**: Desacoplamiento temporal y espacial entre servicios
2. **Kafka**: Streams distribuidos, partitions para escalabilidad
3. **Event Sourcing**: Eventos como single source of truth
4. **Saga**: Transacciones distribuidas con compensación
5. **Idempotencia**: Procesar mensajes múltiples veces sin efectos secundarios
6. **Dead Letter Queues**: Manejo de mensajes que fallan persistentemente
7. **Eventual Consistency**: Trade-off entre consistencia y disponibilidad
8. **Versionado**: Evolución de eventos sin romper consumers existentes
