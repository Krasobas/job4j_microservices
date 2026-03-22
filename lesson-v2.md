# Микросервисная архитектура: полный разбор проекта

## О чём этот урок

Мы строим систему из двух сервисов, которые общаются через Kafka. Звучит просто, но по дороге столкнёмся с кучей вопросов: как два сервиса находят друг друга в Docker-сети? Зачем гонять данные через брокер, если можно просто вызвать HTTP? Как из асинхронного Kafka сделать синхронный запрос-ответ? Как фотографии студентов доходят до браузера, не проходя через Kafka?

Каждый раздел начинается с проблемы, потом — решение с объяснением, потом — код. Не наоборот.

---

## Архитектура: что с чем разговаривает и почему

```
                                    Docker network
┌──────────┐     REST/JSON    ┌─────────────┐   Kafka    ┌─────────────┐
│ Браузер  │ ◄──────────────► │  Service R  │ ◄────────► │  Service S  │
│          │    порт 8080     │  (шлюз)     │            │  (данные)   │
└──────────┘                  └─────────────┘            └──┬──────┬───┘
                                    │                       │      │
                                    │ HTTP (фото)           │      │
                                    └───────────────────────┘      │
                                                            ┌──────┴──────┐
                                                            │             │
                                                       ┌────┴────┐  ┌────┴────┐
                                                       │Postgres │  │  MinIO  │
                                                       │(данные) │  │ (фото)  │
                                                       └─────────┘  └─────────┘
```

### Почему два сервиса, а не один?

В реальном мире разделение на сервисы — это про масштабирование и изоляцию. Service S владеет данными студентов. Service R — шлюз для внешнего мира. Если завтра нужно добавить мобильное приложение — оно подключится к Service R, а Service S не изменится.

В нашем проекте разделение нужно ещё и для демонстрации межсервисного взаимодействия через Kafka.

### Почему Kafka между сервисами, а не прямой HTTP?

Честный ответ: для нашего задания Kafka — overkill. Прямой HTTP-вызов был бы проще и быстрее. Но задание требует брокер, и это проверяет важный навык: умение работать с асинхронным обменом сообщениями.

В реальных системах Kafka между сервисами оправдан когда: нужна буферизация (Service S может быть временно недоступен), нужен fan-out (несколько сервисов читают одни и те же события), нужна надёжная доставка (сообщение не потеряется).

### Почему фото идут через HTTP, а не через Kafka?

Kafka не предназначен для передачи бинарных данных. Максимальный размер сообщения по умолчанию — 1 МБ. Фото может быть 5-10 МБ. Да и зачем? Когда браузер хочет показать фото, он делает отдельный HTTP-запрос. Service R проксирует его к Service S, Service S берёт файл из MinIO и стримит обратно. Kafka в этой цепочке не участвует.

---

## Карта проекта: что где лежит

Прежде чем писать код, нужно понимать, куда что класть. Вот полная структура обоих сервисов:

```
project-root/
├── compose.yaml                              ← Этап 1
│
├── service-r/                                ← ШЛЮЗ (точка входа для браузера)
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/
│       ├── java/ru/job4j/r/
│       │   ├── ServiceRApplication.java
│       │   ├── config/
│       │   │   ├── KafkaConfig.java          ← Этап 3 (ReplyingKafkaTemplate)
│       │   │   ├── CamelRouteConfig.java     ← Этап 8 (Apache Camel)
│       │   │   ├── SecurityConfig.java       ← Этап 9
│       │   │   └── soap/
│       │   │       └── WebServiceConfig.java ← Этап 4 (SOAP на R)
│       │   ├── controller/
│       │   │   └── StudentRestController.java ← Этап 4 (REST API)
│       │   ├── soap/
│       │   │   └── StudentSoapEndpoint.java  ← Этап 4 (SOAP endpoint на R)
│       │   └── service/
│       │       ├── StudentRequestService.java ← Этап 3 (Kafka request-reply)
│       │       ├── XmlToJsonTransformer.java  ← Этап 4 (XML→JSON)
│       │       └── JsonToXmlTransformer.java  ← Этап 4 (JSON→XML)
│       └── resources/
│           ├── application.yaml
│           └── xsd/
│               └── students.xsd              ← Та же XSD, что и в Service S
│
└── service-s/                                ← ДАННЫЕ (PostgreSQL + MinIO + SOAP)
    ├── Dockerfile
    ├── pom.xml
    └── src/main/
        ├── java/ru/job4j/s/
        │   ├── ServiceSApplication.java
        │   ├── config/
        │   │   ├── minio/
        │   │   │   └── MinioConfig.java      ← Этап 5
        │   │   └── soap/
        │   │       └── WebServiceConfig.java  ← Этап 2 (SOAP)
        │   ├── model/
        │   │   └── Student.java              ← Этап 2 (JPA-сущность)
        │   ├── repository/
        │   │   └── StudentRepository.java    ← Этап 2
        │   ├── service/
        │   │   ├── StudentService.java       ← Этап 2 (бизнес-логика)
        │   │   ├── StudentNotFoundException.java
        │   │   └── PhotoService.java         ← Этап 5 (MinIO)
        │   ├── controller/
        │   │   └── InternalPhotoController.java ← Этап 5 (раздача фото)
        │   ├── kafka/
        │   │   └── StudentKafkaListener.java ← Этап 3 (Kafka consumer)
        │   └── soap/
        │       ├── StudentEndpoint.java      ← Этап 2 (SOAP endpoint)
        │       ├── JaxbMarshaller.java       ← Этап 3 (Java→XML)
        │       └── gen/                      ← Генерируется из XSD
        │           ├── GetAllStudentsRequest.java
        │           ├── GetAllStudentsResponse.java
        │           ├── GetStudentRequest.java
        │           ├── GetStudentResponse.java
        │           └── StudentType.java
        └── resources/
            ├── application.yaml
            ├── xsd/
            │   └── students.xsd              ← Этап 2 (SOAP-контракт)
            └── db/changelog/
                ├── db.changelog-master.yaml   ← Liquibase
                └── migrations/
                    └── 001-create-students-table.yaml
```

**Правило: если класс работает с данными (БД, MinIO, SOAP, Kafka listener) — он в Service S. Если класс работает с пользователем (REST, Security, Kafka producer, XML→JSON) — он в Service R.**

---

## Этап 1. Инфраструктура — compose.yaml

### Проблема

У нас 5 компонентов: два сервиса, Kafka, PostgreSQL, MinIO. Каждый — отдельный процесс со своими настройками, портами, зависимостями. Запускать вручную — безумие. И ещё порядок важен: Service S не может стартовать раньше PostgreSQL.

### Решение

Docker Compose описывает все компоненты в одном файле. `docker compose up` поднимает всё. `docker compose down` убивает. Порядок запуска контролируется через health checks.

```yaml
services:

  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: students_db
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d students_db"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**Зачем healthcheck?** Без него Docker считает контейнер "готовым" сразу после запуска процесса. Но PostgreSQL нужно время, чтобы инициализировать базу. `pg_isready` проверяет, что БД реально принимает подключения. Другие контейнеры могут ждать этого события.

**Зачем volume `pg_data`?** Без него данные живут только пока контейнер работает. `docker compose down` — и база пустая. Volume хранит данные на диске хоста, между рестартами они сохраняются.

```yaml
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**Два порта MinIO** — 9000 для S3 API (программный доступ), 9001 для веб-консоли (браузерный интерфейс). В проде веб-консоль не пробрасываем наружу, для разработки удобно.

```yaml
  minio-init:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minioadmin minioadmin;
      mc mb --ignore-existing local/students;
      echo 'MinIO initialized';
      "
```

**Зачем init-контейнер?** MinIO стартует пустым — ни одного бакета. Init-контейнер создаёт бакет `students` и завершается. `--ignore-existing` делает команду безопасной: при повторном запуске не упадёт с "bucket already exists".

```yaml
  kafka:
    image: apache/kafka:3.9.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**KRaft-режим** — Kafka без Zookeeper. До 2023 года Kafka требовала отдельный кластер Zookeeper для хранения метаданных. KRaft убрал эту зависимость. `KAFKA_PROCESS_ROLES: broker,controller` — один узел выполняет обе роли.

**`KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"`** — топики `student-request` и `student-response` создаются автоматически при первом обращении. В проде топики создают заранее с настроенными партициями, но для учебного проекта автосоздание удобнее.

```yaml
  service-s:
    build: ./service-s
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_healthy
      kafka:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/students_db
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: secret
      MINIO_URL: http://minio:9000
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092

  service-r:
    build: ./service-r
    ports:
      - "8080:8080"
    depends_on:
      kafka:
        condition: service_healthy
      service-s:
        condition: service_started
    environment:
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SERVICE_S_URL: http://service-s:8081
      SPRING_THREADS_VIRTUAL_ENABLED: true

volumes:
  pg_data:
  minio_data:
```

**`depends_on` с `condition: service_healthy`** — ключевая вещь. Без неё контейнеры стартуют одновременно, Service S пытается подключиться к PostgreSQL, который ещё не готов, и падает.

**`service-r` → `service-s: condition: service_started`** — не `service_healthy`, потому что у Service S нет healthcheck в compose (он внутренний, healthcheck через Actuator). `service_started` означает "подожди, пока процесс запустится".

**Только Service R пробрасывает порт наружу** (`ports: "8080:8080"`). Это единственная точка входа. PostgreSQL, MinIO, Kafka, Service S — всё живёт внутри Docker-сети и снаружи недоступно.

---

## Этап 2. Service S — данные и SOAP

### Что делает Service S

Владеет данными студентов. Хранит записи в PostgreSQL, фотографии в MinIO. Предоставляет два способа получить данные: SOAP-интерфейс (по условию задачи) и Kafka-слушатель (для взаимодействия с Service R).

### Модель данных

Тут всё просто — JPA-сущность и Spring Data репозиторий:

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/model/Student.java`

```java
@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "record_book_number", unique = true, nullable = false)
    private String recordBookNumber;

    @Column(nullable = false)
    private String faculty;

    @Column(name = "last_name", nullable = false)
    private String lastName;

    @Column(name = "first_name", nullable = false)
    private String firstName;

    @Column(name = "middle_name")
    private String middleName;

    @Column(name = "photo_key")
    private String photoKey;

    /* getters/setters */
}
```

`photoKey` — это не само фото, а ключ файла в MinIO (`photo_001.jpg`). Хранить бинарные данные в БД — антипаттерн.

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/repository/StudentRepository.java`

```java
public interface StudentRepository extends JpaRepository<Student, Long> {
    Optional<Student> findByRecordBookNumber(String recordBookNumber);
}
```

### Создание таблицы через Liquibase

Таблицу не создаём руками. Liquibase делает это автоматически при старте приложения. Подробности — в отдельном уроке по Liquibase. Ключевое: `ddl-auto: validate` в `application.yaml` означает "Hibernate проверяет, что таблица существует, но не создаёт её сам". Создание — ответственность Liquibase.

### SOAP: зачем и как

SOAP — это legacy-протокол обмена XML-сообщениями. Подробный разбор — в отдельном уроке по SOAP. Здесь — только суть.

Spring WS работает по принципу Contract-First: сначала описываешь XSD-схему (контракт), потом генерируешь Java-классы из неё. XSD живёт в `src/main/resources/xsd/students.xsd` и описывает четыре элемента: `getAllStudentsRequest`, `getAllStudentsResponse`, `getStudentRequest`, `getStudentResponse`.

Из XSD плагин `jaxb2-maven-plugin` генерирует классы в пакете `soap.gen`: `GetAllStudentsRequest`, `GetAllStudentsResponse`, `StudentType` и т.д. Эти классы умеют превращаться в XML и обратно.

### StudentService — мост между JPA и SOAP

В проекте две модели: JPA-сущность `Student` (для БД) и JAXB-тип `StudentType` (для XML). Они похожи, но это разные классы с разными аннотациями. `StudentService` маппит одну в другую:

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/service/StudentService.java`

```java
@Service
public class StudentService {

    private final StudentRepository repository;

    public StudentService(StudentRepository repository) {
        this.repository = repository;
    }

    public List<Student> findAll() {
        return repository.findAll();
    }

    public Student findByRecordBookNumber(String number) {
        return repository.findByRecordBookNumber(number)
                .orElseThrow(() -> new StudentNotFoundException(number));
    }

    /**
     * Для SOAP и Kafka — возвращает JAXB-объект,
     * готовый к маршаллингу в XML.
     */
    public GetAllStudentsResponse getAllStudentsAsXml() {
        var response = new GetAllStudentsResponse();
        findAll().stream()
                .map(this::toStudentType)
                .forEach(response.getStudent()::add);
        return response;
    }

    public GetStudentResponse getStudentAsXml(String recordBookNumber) {
        var response = new GetStudentResponse();
        response.setStudent(
                toStudentType(findByRecordBookNumber(recordBookNumber)));
        return response;
    }

    private StudentType toStudentType(Student student) {
        var type = new StudentType();
        type.setRecordBookNumber(student.getRecordBookNumber());
        type.setFaculty(student.getFaculty());
        type.setLastName(student.getLastName());
        type.setFirstName(student.getFirstName());
        type.setMiddleName(student.getMiddleName());
        type.setPhotoKey(student.getPhotoKey());
        return type;
    }
}
```

**Почему два набора методов?** `findAll()` возвращает JPA-сущности — это для внутреннего использования (REST, если понадобится). `getAllStudentsAsXml()` возвращает JAXB-типы — это для SOAP и Kafka, где нужен XML.

### SOAP Endpoint

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/soap/StudentEndpoint.java`

```java
@Endpoint
public class StudentEndpoint {

    private static final String NAMESPACE = "http://job4j.ru/services/students";
    private final StudentService studentService;

    public StudentEndpoint(StudentService studentService) {
        this.studentService = studentService;
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "getAllStudentsRequest")
    @ResponsePayload
    public GetAllStudentsResponse getAll(
            @RequestPayload GetAllStudentsRequest request) {
        return studentService.getAllStudentsAsXml();
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "getStudentRequest")
    @ResponsePayload
    public GetStudentResponse getOne(
            @RequestPayload GetStudentRequest request) {
        return studentService.getStudentAsXml(request.getRecordBookNumber());
    }
}
```

Endpoint — это как REST-контроллер, но маппинг идёт не по URL, а по содержимому XML. `@PayloadRoot` говорит: "если в теле SOAP-запроса корневой элемент `getStudentRequest` из namespace `http://job4j.ru/services/students` — вызови этот метод". Spring WS автоматически десериализует XML в Java-объект (`@RequestPayload`) и сериализует ответ обратно (`@ResponsePayload`).

### WebServiceConfig — инфраструктура SOAP на Service S

Endpoint сам по себе не работает. Нужен сервлет, который принимает HTTP-запросы с SOAP-конвертами, и конфигурация для генерации WSDL из XSD.

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/config/soap/WebServiceConfig.java`

```java
@EnableWs
@Configuration
public class WebServiceConfig {

    /**
     * Сервлет для SOAP-запросов.
     * Аналог DispatcherServlet, но для SOAP.
     * Все запросы на /ws/* обрабатываются Spring WS.
     */
    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(
            ApplicationContext ctx) {
        var servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(ctx);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }

    /**
     * Имя бина "students" определяет URL:
     * WSDL будет доступен по http://host:port/ws/students.wsdl
     */
    @Bean(name = "students")
    public DefaultWsdl11Definition studentsWsdl(XsdSchema studentsSchema) {
        var definition = new DefaultWsdl11Definition();
        definition.setPortTypeName("StudentsPort");
        definition.setLocationUri("/ws");
        definition.setTargetNamespace("http://job4j.ru/services/students");
        definition.setSchema(studentsSchema);
        return definition;
    }

    @Bean
    public XsdSchema studentsSchema() {
        return new SimpleXsdSchema(new ClassPathResource("xsd/students.xsd"));
    }
}
```

**`setTransformWsdlLocations(true)`** — без этого в WSDL будет захардкожен `http://localhost:8081/ws`. С этим флагом Spring WS подставляет адрес из входящего запроса. Критично для деплоя не на localhost.

---

## Этап 3. Kafka — связь между сервисами

### Проблема: как из асинхронного сделать синхронный

Kafka — асинхронный брокер. Ты кидаешь сообщение в топик и забываешь. Это как отправить письмо — ты не стоишь у почтового ящика и не ждёшь ответа.

Но пользователь делает `GET /api/students` и ждёт JSON прямо сейчас. Браузер висит и ждёт. Нам нужен паттерн request-reply: отправить запрос через Kafka и дождаться ответа.

### Как request-reply работает

Представь двух людей в разных комнатах. Они общаются через ящики:

1. **Service R** пишет записку "дай всех студентов", приклеивает стикер с номером `#42` и кладёт в ящик `student-request`.
2. **Service S** берёт записку, готовит ответ, переклеивает стикер `#42` на ответ и кладёт в ящик `student-response`.
3. **Service R** сидит у ящика `student-response`, видит ответ со стикером `#42` — "это мой!" — и забирает.

Стикер с номером — это **correlation ID**. Без него Service R не отличит свой ответ от чужого.

### ReplyingKafkaTemplate — Spring делает это за тебя

Spring Kafka написал всю эту механику: генерацию correlation ID, ожидание ответа, маршрутизацию по ID. Тебе остаётся вызвать один метод.

Но template нужно настроить. Ему нужны две вещи:

**ProducerFactory** — чтобы отправлять запросы. Стандартная настройка, ничего особенного.

**Reply container** — слушатель на топике ответов. Обычный `@KafkaListener` не подходит, потому что template должен сам управлять маршрутизацией ответов по correlation ID. Поэтому создаём контейнер программно:

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/config/KafkaConfig.java`

```java
@Configuration
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(Map.of(
                ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers,
                ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class,
                ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class
        ));
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(Map.of(
                ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers,
                ConsumerConfig.GROUP_ID_CONFIG, "service-r",
                ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class,
                ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class,
                ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"
        ));
    }

    @Bean
    public ConcurrentMessageListenerContainer<String, String> replyContainer(
            ConsumerFactory<String, String> cf) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(cf);
        /*
         * Этот контейнер слушает топик student-response.
         * Все ответы от Service S приходят сюда.
         * Template подключится к нему и будет маршрутизировать
         * ответы по correlation ID.
         */
        var container = factory.createContainer("student-response");
        /* Своя consumer group, чтобы не конфликтовать с другими слушателями */
        container.getContainerProperties().setGroupId("service-r-reply");
        /* Template сам запустит контейнер когда будет готов */
        container.setAutoStartup(false);
        return container;
    }

    @Bean
    public ReplyingKafkaTemplate<String, String, String> replyingKafkaTemplate(
            ProducerFactory<String, String> pf,
            ConcurrentMessageListenerContainer<String, String> replyContainer) {
        var template = new ReplyingKafkaTemplate<>(pf, replyContainer);
        /* Если Service S не ответил за 10 секунд — таймаут */
        template.setDefaultReplyTimeout(Duration.ofSeconds(10));
        /*
         * Один топик ответов на всех. Без этого флага template
         * будет ожидать отдельный топик для каждого инстанса Service R
         * и начнёт ругаться в логах на "чужие" ответы.
         */
        template.setSharedReplyTopic(true);
        return template;
    }
}
```

### Service R: отправка запроса

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/service/StudentRequestService.java`

```java
@Service
public class StudentRequestService {

    private static final Logger LOG = LoggerFactory.getLogger(StudentRequestService.class);
    private static final String REQUEST_TOPIC = "student-request";

    private final ReplyingKafkaTemplate<String, String, String> replyingTemplate;

    public StudentRequestService(
            ReplyingKafkaTemplate<String, String, String> replyingTemplate) {
        this.replyingTemplate = replyingTemplate;
    }

    /**
     * Отправляет запрос в Kafka и ждёт ответа.
     *
     * @param payload "ALL" для всех студентов,
     *                или recordBookNumber для одного
     * @return XML-строка с ответом от Service S
     */
    public String sendAndReceive(String payload) {
        LOG.info("Sending request: {}", payload);

        var record = new ProducerRecord<String, String>(REQUEST_TOPIC, payload);
        /*
         * Говорим Service S, куда класть ответ.
         * Без этого заголовка Service S не знает, в какой топик писать.
         */
        record.headers().add(KafkaHeaders.REPLY_TOPIC,
                "student-response".getBytes(StandardCharsets.UTF_8));
        try {
            /*
             * Одна строка вместо всей ручной механики.
             * Внутри: генерирует correlation ID, отправляет сообщение,
             * блокируется на ответе (или таймауте).
             */
            RequestReplyFuture<String, String, String> future =
                    replyingTemplate.sendAndReceive(record);
            ConsumerRecord<String, String> response = future.get().value();
            LOG.info("Got response from Service S");
            return response.value();
        } catch (Exception e) {
            throw new RuntimeException("Service S did not respond", e);
        }
    }
}
```

### Service S: обработка запроса и ответ

Service S не знает про `ReplyingKafkaTemplate`. Он просто слушает топик, формирует ответ и **обязательно копирует correlation ID**. Если забудет — template на стороне R не сопоставит ответ с запросом и будет таймаут.

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/kafka/StudentKafkaListener.java`

```java
@Component
public class StudentKafkaListener {

    private static final Logger LOG = LoggerFactory.getLogger(StudentKafkaListener.class);

    private final StudentService studentService;
    private final JaxbMarshaller marshaller;
    private final KafkaTemplate<String, String> kafkaTemplate;

    public StudentKafkaListener(StudentService studentService,
                                 JaxbMarshaller marshaller,
                                 KafkaTemplate<String, String> kafkaTemplate) {
        this.studentService = studentService;
        this.marshaller = marshaller;
        this.kafkaTemplate = kafkaTemplate;
    }

    @KafkaListener(topics = "student-request", groupId = "service-s")
    public void handle(
            @Payload String payload,
            @Header(KafkaHeaders.REPLY_TOPIC) byte[] replyTopic,
            @Header(KafkaHeaders.CORRELATION_ID) byte[] correlationId) {

        LOG.info("Received request: {}", payload);

        /*
         * Протокол простой:
         * "ALL" — вернуть всех студентов
         * Любая другая строка — recordBookNumber конкретного студента
         */
        Object xmlObject;
        if ("ALL".equals(payload)) {
            xmlObject = studentService.getAllStudentsAsXml();
        } else {
            xmlObject = studentService.getStudentAsXml(payload);
        }

        /* Java-объект → XML-строка через JAXB */
        String xmlResponse = marshaller.marshal(xmlObject);

        /*
         * Отправляем ответ в топик, который указал Service R.
         * Это не хардкод — Service R передал имя топика в заголовке.
         */
        var record = new ProducerRecord<String, String>(
                new String(replyTopic, StandardCharsets.UTF_8),
                xmlResponse);

        /*
         * КРИТИЧНО: копируем correlation ID.
         * Без этого ReplyingKafkaTemplate не найдёт ответ
         * и через 10 секунд выбросит TimeoutException.
         */
        record.headers().add(KafkaHeaders.CORRELATION_ID, correlationId);

        kafkaTemplate.send(record);
        LOG.info("Response sent");
    }
}
```

### JaxbMarshaller — конвертация Java → XML

Spring WS делает маршаллинг автоматически для SOAP Endpoint. Но для Kafka нам нужна XML-строка, а не Java-объект. Поэтому маршаллим вручную:

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/soap/JaxbMarshaller.java`

```java
@Component
public class JaxbMarshaller {

    private final JAXBContext context;

    /**
     * JAXBContext — тяжёлый объект.
     * Создаём один раз при старте, переиспользуем всегда.
     * Он потокобезопасный.
     */
    public JaxbMarshaller() throws JAXBException {
        this.context = JAXBContext.newInstance(
                GetAllStudentsResponse.class,
                GetStudentResponse.class
        );
    }

    public String marshal(Object object) {
        try {
            /*
             * Marshaller — НЕ потокобезопасный.
             * Создаём новый на каждый вызов.
             * Это дёшево, в отличие от JAXBContext.
             */
            Marshaller marshaller = context.createMarshaller();
            marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
            var writer = new StringWriter();
            marshaller.marshal(object, writer);
            return writer.toString();
        } catch (JAXBException e) {
            throw new RuntimeException("Failed to marshal", e);
        }
    }
}
```

### Полная картина: что происходит при Kafka-запросе

```
Service R                          Kafka                         Service S
    │                                │                                │
    │  sendAndReceive("ALL")         │                                │
    │  template генерирует           │                                │
    │  correlationId: abc-123        │                                │
    │                                │                                │
    │  → student-request             │                                │
    │  headers:                      │                                │
    │    REPLY_TOPIC: student-resp   │                                │
    │    CORRELATION_ID: abc-123     │                                │
    │──────────────────────────────► │                                │
    │                                │ ──────────────────────────────►│
    │                                │                                │
    │  future.get() — ждёт...        │   @KafkaListener получает     │
    │                                │   payload: "ALL"              │
    │                                │   replyTopic: student-resp    │
    │                                │   correlationId: abc-123      │
    │                                │                                │
    │                                │   → StudentService.getAll()   │
    │                                │   → JAXB marshal → XML        │
    │                                │                                │
    │                                │ ◄──────────────────────────────│
    │                                │   ← student-response          │
    │                                │   headers:                    │
    │                                │     CORRELATION_ID: abc-123   │
    │◄────────────────────────────── │                                │
    │                                │                                │
    │  template видит abc-123        │                                │
    │  → future.complete(XML)        │                                │
    │  → sendAndReceive возвращает   │                                │
```

---

## Этап 4. Service R — REST и трансформация

### Проблема: XML → JSON

Service S отвечает XML (потому что SOAP). Но пользователь ждёт JSON (потому что REST). Service R должен трансформировать.

### XmlToJsonTransformer

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/service/XmlToJsonTransformer.java`

```java
@Service
public class XmlToJsonTransformer {

    private static final Logger LOG = LoggerFactory.getLogger(XmlToJsonTransformer.class);

    private final XmlMapper xmlMapper = new XmlMapper();
    private final ObjectMapper jsonMapper = new ObjectMapper();

    /**
     * Конвертирует XML-строку от Service S в JSON.
     * Дополнительно подменяет photoKey (внутренний ключ MinIO)
     * на photoUrl (публичный URL для браузера).
     */
    public String xmlToJson(String xml) {
        try {
            JsonNode node = xmlMapper.readTree(xml);
            addPhotoUrls(node);
            return jsonMapper.writerWithDefaultPrettyPrinter()
                    .writeValueAsString(node);
        } catch (Exception e) {
            throw new RuntimeException("XML to JSON failed", e);
        }
    }

    private void addPhotoUrls(JsonNode node) {
        if (node.has("student")) {
            JsonNode students = node.get("student");
            if (students.isArray()) {
                students.forEach(this::replacePhotoKey);
            } else {
                replacePhotoKey(students);
            }
        }
    }

    /**
     * photoKey — это внутренний ключ в MinIO (photo_001.jpg).
     * Браузер не имеет доступа к MinIO.
     * Подменяем на URL, через который Service R проксирует фото.
     */
    private void replacePhotoKey(JsonNode student) {
        if (student instanceof ObjectNode obj && obj.has("photoKey")) {
            String rbNumber = obj.get("recordBookNumber").asText();
            obj.put("photoUrl", "/api/students/" + rbNumber + "/photo");
            obj.remove("photoKey");
        }
    }
}
```

**Зачем подменять photoKey на photoUrl?** `photoKey = "photo_001.jpg"` — это внутреннее имя файла в MinIO. Браузер ничего не может с ним сделать. `photoUrl = "/api/students/RB-001/photo"` — это публичный URL, по которому браузер может запросить фото у Service R. MinIO скрыт от внешнего мира.

### REST-контроллер

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/controller/StudentRestController.java`

```java
@RestController
@RequestMapping("/api/students")
public class StudentRestController {

    private static final Logger LOG = LoggerFactory.getLogger(StudentRestController.class);

    private final StudentRequestService requestService;
    private final XmlToJsonTransformer transformer;
    private final RestClient restClient;

    public StudentRestController(
            StudentRequestService requestService,
            XmlToJsonTransformer transformer,
            @Value("${service-s.url}") String serviceSUrl) {
        this.requestService = requestService;
        this.transformer = transformer;
        this.restClient = RestClient.builder()
                .baseUrl(serviceSUrl)
                .build();
    }

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> getAllStudents() {
        LOG.info("GET /api/students");

        /* 1. Отправляем "ALL" в Kafka, ждём XML-ответ */
        String xml = requestService.sendAndReceive("ALL");

        /* 2. XML → JSON + подмена photoKey → photoUrl */
        String json = transformer.xmlToJson(xml);

        /* 3. Возвращаем JSON клиенту */
        LOG.info("Returning JSON to client");
        return ResponseEntity.ok(json);
    }

    @GetMapping(value = "/{recordBookNumber}",
                produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> getStudent(
            @PathVariable String recordBookNumber) {
        LOG.info("GET /api/students/{}", recordBookNumber);
        String xml = requestService.sendAndReceive(recordBookNumber);
        String json = transformer.xmlToJson(xml);
        return ResponseEntity.ok(json);
    }

    /**
     * Фото не ходит через Kafka.
     * Service R проксирует запрос напрямую к Service S по HTTP.
     */
    @GetMapping(value = "/{recordBookNumber}/photo",
                produces = MediaType.IMAGE_JPEG_VALUE)
    public ResponseEntity<byte[]> getPhoto(
            @PathVariable String recordBookNumber) {
        LOG.info("GET /api/students/{}/photo — proxying", recordBookNumber);

        /*
         * Сначала узнаём photoKey через Kafka (или кэш в будущем).
         * Потом запрашиваем файл у Service S по HTTP.
         */
        String xml = requestService.sendAndReceive(recordBookNumber);
        String photoKey = extractPhotoKey(xml);
        if (photoKey == null) {
            return ResponseEntity.notFound().build();
        }

        byte[] photo = restClient.get()
                .uri("/internal/photos/{key}", photoKey)
                .retrieve()
                .body(byte[].class);

        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_JPEG)
                .body(photo);
    }

    private String extractPhotoKey(String xml) {
        int start = xml.indexOf("<photoKey>");
        int end = xml.indexOf("</photoKey>");
        if (start == -1 || end == -1) {
            return null;
        }
        return xml.substring(start + "<photoKey>".length(), end);
    }
}
```

### Как фото доходит до браузера

```
Браузер                    Service R                  Service S              MinIO
   │                          │                          │                     │
   │ GET /api/students        │                          │                     │
   │─────────────────────────►│                          │                     │
   │                          │ Kafka: "ALL"             │                     │
   │                          │─────────────────────────►│                     │
   │                          │◄─────────────────────────│                     │
   │                          │ XML→JSON                 │                     │
   │◄─────────────────────────│                          │                     │
   │                          │                          │                     │
   │ JSON:                    │                          │                     │
   │  photoUrl: /api/.../photo│                          │                     │
   │                          │                          │                     │
   │ GET /api/.../photo       │                          │                     │
   │─────────────────────────►│                          │                     │
   │                          │ HTTP: /internal/photos/  │                     │
   │                          │─────────────────────────►│                     │
   │                          │                          │ getObject()         │
   │                          │                          │────────────────────►│
   │                          │                          │◄────────────────────│
   │                          │◄─────────────────────────│ byte[]              │
   │◄─────────────────────────│ byte[]                   │                     │
   │ показывает фото          │                          │                     │
```

Два отдельных запроса. Данные студентов — через Kafka (как требует задание). Фото — прямой HTTP между сервисами (потому что бинарные данные через Kafka гонять нельзя).

### JSON → XML трансформация

Задание требует трансформацию в обе стороны: XML→JSON и JSON→XML. XML→JSON мы уже сделали для REST. JSON→XML нужен для SOAP-интерфейса Service R: внешняя система шлёт JSON, Service R должен уметь отдать XML.

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/service/JsonToXmlTransformer.java`

```java
@Service
public class JsonToXmlTransformer {

    private final ObjectMapper jsonMapper = new ObjectMapper();
    private final XmlMapper xmlMapper = new XmlMapper();

    /**
     * Конвертирует JSON-строку в XML.
     * Используется в SOAP-интерфейсе Service R.
     */
    public String jsonToXml(String json) {
        try {
            JsonNode node = jsonMapper.readTree(json);
            return xmlMapper.writerWithDefaultPrettyPrinter()
                    .writeValueAsString(node);
        } catch (Exception e) {
            throw new RuntimeException("JSON to XML failed", e);
        }
    }
}
```

### SOAP-интерфейс Service R

Задание говорит: Service R имеет два интерфейса — REST и SOAP. REST мы сделали. SOAP-интерфейс на Service R дублирует тот же функционал, но для систем, которые работают только по SOAP.

Логика: SOAP-запрос приходит на Service R → идёт в Kafka → получает XML от Service S → возвращает XML клиенту. Трансформация XML→JSON **не нужна** — SOAP-клиент ожидает XML.

Service R использует **ту же XSD-схему**, что и Service S. Из неё генерируются те же JAXB-классы. Оба сервиса "говорят на одном языке".

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/config/soap/WebServiceConfig.java`

```java
@EnableWs
@Configuration
public class WebServiceConfig {

    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(
            ApplicationContext ctx) {
        var servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(ctx);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }

    @Bean(name = "students")
    public DefaultWsdl11Definition studentsWsdl(XsdSchema studentsSchema) {
        var definition = new DefaultWsdl11Definition();
        definition.setPortTypeName("StudentsPort");
        definition.setLocationUri("/ws");
        definition.setTargetNamespace("http://job4j.ru/services/students");
        definition.setSchema(studentsSchema);
        return definition;
    }

    @Bean
    public XsdSchema studentsSchema() {
        return new SimpleXsdSchema(new ClassPathResource("xsd/students.xsd"));
    }
}
```

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/soap/StudentSoapEndpoint.java`

```java
@Endpoint
public class StudentSoapEndpoint {

    private static final String NAMESPACE = "http://job4j.ru/services/students";

    private final StudentRequestService requestService;
    private final JaxbMarshaller marshaller;

    public StudentSoapEndpoint(StudentRequestService requestService) {
        this.requestService = requestService;
        /*
         * На стороне R тоже нужен unmarshaller для разбора XML-ответа
         * от Service S и формирования JAXB-объекта для SOAP-ответа.
         * Но Service S уже возвращает готовый XML — можно передать как есть.
         */
    }

    /**
     * SOAP-клиент шлёт запрос на Service R.
     * Service R идёт в Kafka → получает XML от Service S
     * → разбирает его → возвращает SOAP-ответ.
     */
    @PayloadRoot(namespace = NAMESPACE, localPart = "getAllStudentsRequest")
    @ResponsePayload
    public GetAllStudentsResponse getAll(
            @RequestPayload GetAllStudentsRequest request) {
        String xml = requestService.sendAndReceive("ALL");
        return unmarshal(xml, GetAllStudentsResponse.class);
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "getStudentRequest")
    @ResponsePayload
    public GetStudentResponse getOne(
            @RequestPayload GetStudentRequest request) {
        String xml = requestService.sendAndReceive(request.getRecordBookNumber());
        return unmarshal(xml, GetStudentResponse.class);
    }

    /**
     * XML-строка от Service S → Java-объект для Spring WS.
     * Spring WS сам сериализует его обратно в XML для SOAP-ответа.
     */
    private <T> T unmarshal(String xml, Class<T> clazz) {
        try {
            var context = JAXBContext.newInstance(clazz);
            var unmarshaller = context.createUnmarshaller();
            return clazz.cast(unmarshaller.unmarshal(
                    new StringReader(xml)));
        } catch (JAXBException e) {
            throw new RuntimeException("Failed to unmarshal XML", e);
        }
    }
}
```

**Разница между REST и SOAP на Service R:**
- REST-контроллер: Kafka → XML → **трансформирует в JSON** → возвращает клиенту
- SOAP-endpoint: Kafka → XML → **разбирает в Java-объект** → Spring WS оборачивает обратно в XML → возвращает клиенту

Оба получают одни и те же данные из Kafka, но отдают в разных форматах.

**Что нужно в pom.xml Service R для SOAP:** те же зависимости, что в Service S — `spring-boot-starter-web-services`, `wsdl4j`, `jakarta.xml.bind-api`, `jaxb-runtime`. И тот же `jaxb2-maven-plugin` с той же XSD. Скопируй `students.xsd` в `service-r/src/main/resources/xsd/` — оба сервиса используют один контракт, но каждый генерирует свои JAXB-классы независимо. Также нужна зависимость `jackson-dataformat-xml` для `XmlMapper` в трансформерах.

---

## Этап 5. Service S — раздача фото

Service S владеет MinIO. Наружу MinIO не торчит. Для раздачи фото есть внутренний HTTP-endpoint:

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/controller/InternalPhotoController.java`

```java
@RestController
@RequestMapping("/internal/photos")
public class InternalPhotoController {

    private final PhotoService photoService;

    public InternalPhotoController(PhotoService photoService) {
        this.photoService = photoService;
    }

    @GetMapping("/{key}")
    public ResponseEntity<byte[]> getPhoto(@PathVariable String key) {
        byte[] photo = photoService.getPhoto(key);
        if (photo.length == 0) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_JPEG)
                .header(HttpHeaders.CACHE_CONTROL, "max-age=3600")
                .body(photo);
    }
}
```

`PhotoService` берёт файл из MinIO:

> **📁 Service S** → `service-s/src/main/java/ru/job4j/s/service/PhotoService.java`

```java
@Service
public class PhotoService {

    private final MinioClient minioClient;
    private final String bucket;

    public PhotoService(MinioClient minioClient,
                        @Value("${minio.bucket}") String bucket) {
        this.minioClient = minioClient;
        this.bucket = bucket;
    }

    public byte[] getPhoto(String photoKey) {
        try (InputStream stream = minioClient.getObject(
                GetObjectArgs.builder()
                        .bucket(bucket)
                        .object(photoKey)
                        .build())) {
            return stream.readAllBytes();
        } catch (ErrorResponseException e) {
            if ("NoSuchKey".equals(e.errorResponse().code())) {
                return new byte[0];
            }
            throw new RuntimeException("MinIO error", e);
        } catch (Exception e) {
            throw new RuntimeException("Failed to get photo", e);
        }
    }
}
```

**Почему `/internal/photos`?** Слово "internal" — конвенция, сигнал что этот endpoint не для внешних клиентов. В Docker-сети порт Service S не проброшен наружу, так что браузер физически не может до него добраться. Но название напоминает разработчику: "это внутренний API, не документируй его в Swagger".

---

## Этап 6. Virtual Threads

Одна строка в `application.yaml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

**Зачем?** В Service R запрос блокируется на `future.get()`, ожидая ответа из Kafka. С обычными потоками каждый ожидающий запрос занимает поток ОС. 200 одновременных запросов = 200 потоков, которые ничего не делают, просто ждут. С virtual threads (Java 21+) заблокированный поток не занимает ресурсы ОС. 200 ожидающих запросов — почти бесплатно.

Для нашего учебного проекта с тремя студентами разницы не будет. Но это показывает осведомлённость о платформе.

---

## Этап 7. Swagger

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

После подключения зависимости Swagger UI автоматически доступен по `http://localhost:8080/swagger-ui.html`. Он сканирует все `@RestController` и генерирует интерактивную документацию. Можно отправлять запросы прямо из браузера.

Для улучшения документации — аннотации на методах контроллера:

```java
@Operation(summary = "Get all students",
           description = "Fetches all student records via Kafka from Service S")
@GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<String> getAllStudents() { /* ... */ }
```

---

## Этап 8. Apache Camel — маршрутизация и трансформация

### Проблема

В `StudentRestController` прямо сейчас три шага: отправить в Kafka → получить XML → трансформировать в JSON. Для двух эндпоинтов это терпимо. Но представь, что Service R интегрируется с десятью системами: одна отдаёт через Kafka, вторая через FTP, третья через JMS. Каждая интеграция — свой транспорт, свой формат, своя обработка ошибок. Код контроллера превращается в спагетти.

Apache Camel — фреймворк для интеграций. Вместо ручного кода ты описываешь маршрут декларативно: откуда взять данные → как трансформировать → куда отдать. Camel знает 300+ компонентов (Kafka, HTTP, FTP, SMTP, S3 и т.д.), все работают через единый API.

Для двух маршрутов в нашем проекте Camel — overkill. Но задание просит, и на собесе можно объяснить, когда он оправдан, а когда нет.

### Как Camel встраивается в Service R

Вместо того чтобы контроллер напрямую вызывал `StudentRequestService` и `XmlToJsonTransformer`, запрос проходит через Camel-маршрут. Camel берёт на себя цепочку вызовов и логирование.

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/config/CamelRouteConfig.java`

```java
@Component
public class CamelRouteConfig extends RouteBuilder {

    @Override
    public void configure() {

        /*
         * Маршрут вызывается из контроллера через:
         *   producerTemplate.requestBody("direct:getAllStudents", "ALL", String.class)
         *
         * "direct:" — синхронный внутренний endpoint Camel.
         * Просто точка входа в маршрут, без сети и очередей.
         */
        from("direct:getAllStudents")
                .routeId("get-all-students")
                .log("Sending request to Kafka for all students")
                .to("bean:studentRequestService?method=sendAndReceive")
                .log("Got XML response, transforming to JSON")
                .to("bean:xmlToJsonTransformer?method=xmlToJson")
                .log("JSON ready, returning to controller");

        from("direct:getStudent")
                .routeId("get-one-student")
                .log("Sending request to Kafka for student: ${body}")
                .to("bean:studentRequestService?method=sendAndReceive")
                .log("Got XML response, transforming to JSON")
                .to("bean:xmlToJsonTransformer?method=xmlToJson")
                .log("JSON ready, returning to controller");
    }
}
```

**Что тут происходит:**
- `from("direct:getAllStudents")` — точка входа. Контроллер вызывает этот маршрут по имени.
- `.to("bean:studentRequestService?method=sendAndReceive")` — вызывает Spring-бин. Body маршрута (строка `"ALL"`) передаётся как аргумент метода. Результат (XML-строка) становится новым body.
- `.to("bean:xmlToJsonTransformer?method=xmlToJson")` — следующий шаг. Получает XML из предыдущего шага, возвращает JSON.
- `.log(...)` — логирование встроено в маршрут, не в бизнес-код.

### Контроллер с Camel

Контроллер становится совсем тонким:

```java
@RestController
@RequestMapping("/api/students")
public class StudentRestController {

    private final ProducerTemplate producerTemplate;

    public StudentRestController(ProducerTemplate producerTemplate) {
        this.producerTemplate = producerTemplate;
    }

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> getAllStudents() {
        String json = producerTemplate.requestBody(
                "direct:getAllStudents", "ALL", String.class);
        return ResponseEntity.ok(json);
    }

    @GetMapping(value = "/{recordBookNumber}",
                produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> getStudent(
            @PathVariable String recordBookNumber) {
        String json = producerTemplate.requestBody(
                "direct:getStudent", recordBookNumber, String.class);
        return ResponseEntity.ok(json);
    }
}
```

`ProducerTemplate` — Spring-бин, который Camel создаёт автоматически. Через него контроллер "входит" в маршрут.

### Зависимости

```xml
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter</artifactId>
    <version>4.9.0</version>
</dependency>
```

### Когда Camel реально нужен

Для нашего проекта Camel не даёт преимущества — те же три строки, обёрнутые в маршрут. Но в проектах с десятками интеграций (банки, страховые, логистика) Camel резко упрощает жизнь: единый DSL для всех транспортов, встроенный error handling, retry, circuit breaker, трансформация форматов. Поменять Kafka на RabbitMQ — замена одной строки `from("kafka:...")` на `from("rabbitmq:...")`.

---

## Этап 9. Spring Security

### Проблема

Без авторизации любой может вызвать API. В задании требуется Spring Security.

### Минимальная настройка

> **📁 Service R** → `service-r/src/main/java/ru/job4j/r/config/SecurityConfig.java`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                /* Отключаем CSRF — у нас stateless API, не форма */
                .csrf(csrf -> csrf.disable())
                /* Не храним сессию — каждый запрос несёт credentials */
                .sessionManagement(sm ->
                        sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        /* Swagger и healthcheck — без авторизации */
                        .requestMatchers(
                                "/swagger-ui/**",
                                "/api-docs/**",
                                "/v3/api-docs/**",
                                "/actuator/health"
                        ).permitAll()
                        /* Всё остальное — нужен логин */
                        .requestMatchers("/api/**").authenticated()
                        .anyRequest().permitAll()
                )
                /* HTTP Basic — логин:пароль в каждом запросе */
                .httpBasic(Customizer.withDefaults())
                .build();
    }

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        var user = User.builder()
                .username("admin")
                .password(encoder.encode("admin"))
                .roles("USER")
                .build();
        return new InMemoryUserDetailsManager(user);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**HTTP Basic** — простейший механизм. Логин и пароль передаются в заголовке каждого запроса (Base64-кодированные). Для продакшена это не подходит — там OAuth2/JWT. Но для тестового задания — достаточно.

После этого `curl http://localhost:8080/api/students` вернёт 401. А `curl -u admin:admin http://localhost:8080/api/students` — работает.

---

## Этап 10. Dockerfile

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN apt-get update && apt-get install -y maven \
    && mvn clean package -DskipTests \
    && rm -rf /root/.m2

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Multi-stage build** — первый этап (`build`) содержит JDK + Maven + исходники. Второй — только JRE + готовый JAR. Итоговый образ минимальный: нет исходников, нет Maven, нет JDK.

**`eclipse-temurin:21`** — проверенная сборка OpenJDK от Adoptium. Не `openjdk` (deprecated).

---

## Этап 11. Запуск и проверка

```bash
docker compose up --build
```

Проверка:

```bash
# Все студенты (с авторизацией)
curl -u admin:admin http://localhost:8080/api/students | jq

# Один студент
curl -u admin:admin http://localhost:8080/api/students/RB-001 | jq

# Фото
curl -u admin:admin http://localhost:8080/api/students/RB-001/photo -o photo.jpg

# SOAP WSDL
curl http://localhost:8081/ws/students.wsdl

# Swagger UI — открыть в браузере
# http://localhost:8080/swagger-ui.html

# Health check
curl http://localhost:8080/actuator/health
```

Ожидаемый JSON:

```json
[
  {
    "recordBookNumber": "RB-001",
    "faculty": "Computer Science",
    "lastName": "Ivanov",
    "firstName": "Ivan",
    "middleName": "Ivanovich",
    "photoUrl": "/api/students/RB-001/photo"
  }
]
```

---

## Полный сценарий: от запроса до ответа

1. Пользователь открывает `http://localhost:8080/api/students` в браузере.
2. Браузер спрашивает логин/пароль (HTTP Basic). Вводим `admin`/`admin`.
3. Spring Security проверяет credentials. ОК.
4. `StudentRestController.getAllStudents()` вызывается.
5. `StudentRequestService.sendAndReceive("ALL")` отправляет сообщение в Kafka-топик `student-request` с correlation ID и заголовком `REPLY_TOPIC: student-response`.
6. Virtual thread блокируется на `future.get()`, ожидая ответа.
7. **Service S**: `StudentKafkaListener` получает сообщение из `student-request`.
8. Вызывает `StudentService.getAllStudentsAsXml()` → репозиторий достаёт данных из PostgreSQL → маппит `Student` → `StudentType` → `GetAllStudentsResponse`.
9. `JaxbMarshaller` конвертирует Java-объект в XML-строку.
10. Отправляет XML в `student-response` с тем же correlation ID.
11. **Service R**: `ReplyingKafkaTemplate` видит ответ с нужным correlation ID → `future.complete()`.
12. `sendAndReceive()` возвращает XML.
13. `XmlToJsonTransformer` конвертирует XML → JSON, подменяет `photoKey` → `photoUrl`.
14. JSON возвращается пользователю.
15. Браузер рендерит JSON. Если в интерфейсе есть `<img src="/api/students/RB-001/photo">`, браузер делает отдельный GET-запрос.
16. Service R проксирует запрос к `http://service-s:8081/internal/photos/photo_001.jpg`.
17. Service S берёт файл из MinIO, стримит обратно.
18. Service R отдаёт `byte[]` браузеру. Фото отображается.

Все шаги логируются в консоли Service R.
