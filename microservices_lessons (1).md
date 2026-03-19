# Микросервисы — практические уроки (2026)

> **Целевой стек**: Java 21, Spring Boot 3.4, Spring Cloud 2025.0.x (`2025.0.0` + Spring Boot 3.5 / `2024.0.x` + Spring Boot 3.4)
>
> **Предпосылки**: пройдены разделы Docker и Kubernetes. Есть понимание, что такое микросервис, REST, Spring Boot.

---

## Урок 1. Межсервисное взаимодействие (REST)

### Зачем

Микросервисы — это отдельные приложения. Но они должны общаться. Самый простой и частый способ — HTTP-вызовы (REST). В этом уроке разберём, как один Spring Boot сервис вызывает другой.

### Подготовка: два сервиса

Создадим два простых Spring Boot проекта:

**catalog-service** (порт 8081) — хранит товары:

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public List<Product> getAll() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public Product getOne(@PathVariable Long id) {
        return productService.findById(id);
    }
}
```

**order-service** (порт 8082) — при создании заказа должен проверить, что товар существует. Для этого вызывает catalog-service.

### Варианты HTTP-клиентов

| Клиент | Статус в 2026 | Стиль |
|--------|--------------|-------|
| `RestTemplate` | **Deprecated** (с Spring 5.0) | Императивный |
| `WebClient` | Актуален | Реактивный (WebFlux) |
| **`RestClient`** | ✅ Рекомендуется (Spring 6.1+) | Императивный (Spring MVC) |
| **HTTP Interface** | ✅ Рекомендуется (Spring 6+) | Декларативный |
| OpenFeign | Работает, уступает HTTP Interface | Legacy |

### Вариант 1: RestClient (императивный)

```java
@Configuration
public class ClientConfig {

    @Bean
    RestClient catalogRestClient() {
        return RestClient.builder()
                .baseUrl("http://localhost:8081")
                .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class CatalogClient {

    private final RestClient catalogRestClient;

    public Product getProduct(Long id) {
        return catalogRestClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .body(Product.class);
    }

    public List<Product> getAllProducts() {
        return catalogRestClient.get()
                .uri("/api/products")
                .retrieve()
                .body(new ParameterizedTypeReference<>() {});
    }
}
```

RestClient — замена RestTemplate. Тот же синхронный подход, но с fluent API и поддержкой новых фич Spring.

### Вариант 2: HTTP Interface (декларативный)

Это Spring-нативная замена OpenFeign. Вы объявляете интерфейс, Spring генерирует реализацию.

**Шаг 1: Объявить интерфейс**

```java
public interface CatalogClient {

    @GetExchange("/api/products")
    List<Product> getAllProducts();

    @GetExchange("/api/products/{id}")
    Product getProduct(@PathVariable Long id);

    @PostExchange("/api/products")
    Product createProduct(@RequestBody ProductRequest request);
}
```

**Шаг 2: Создать бин**

```java
@Configuration
public class ClientConfig {

    @Bean
    CatalogClient catalogClient(RestClient.Builder builder) {
        RestClient restClient = builder
                .baseUrl("http://localhost:8081")
                .build();
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
                .builderFor(adapter)
                .build();
        return factory.createClient(CatalogClient.class);
    }
}
```

**Шаг 3: Использовать**

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final CatalogClient catalogClient;

    public Order createOrder(Long productId, int quantity) {
        Product product = catalogClient.getProduct(productId);
        if (product == null) {
            throw new ProductNotFoundException(productId);
        }
        return new Order(product.id(), product.name(), quantity, product.price() * quantity);
    }
}
```

### RestClient vs HTTP Interface — что выбрать

| Критерий | RestClient | HTTP Interface |
|---------|-----------|---------------|
| Стиль | Императивный (цепочка вызовов) | Декларативный (интерфейс + аннотации) |
| Гибкость | Полная (headers, query params, error handling в каждом вызове) | Меньше (стандартные сценарии) |
| Читаемость | Код вызова размазан по сервисам | Контракт виден в одном месте |
| Тестируемость | Мокаем RestClient | Мокаем интерфейс (проще) |
| Когда | Сложная логика вызовов, нестандартные headers | Стандартные CRUD-вызовы между сервисами |

**Рекомендация**: HTTP Interface для типовых вызовов. RestClient для сложных случаев (динамические headers, multipart, streaming).

### Проблема: захардкоженный URL

Сейчас мы пишем `http://localhost:8081`. Это работает локально, но:
- В Docker Compose адрес будет `http://catalog-service:8081`.
- В Kubernetes — `http://catalog-service.default.svc.cluster.local`.
- Если catalog-service масштабирован до 3 копий — на какую стучать?

Решение — **Service Discovery** (следующий урок).

### Запуск через Docker Compose

Пока без Service Discovery — просто два сервиса:

```yaml
services:
  catalog-service:
    build: ./catalog-service
    ports:
      - "8081:8081"

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    environment:
      CATALOG_SERVICE_URL: http://catalog-service:8081
    depends_on:
      - catalog-service
```

В order-service подхватываем URL из переменной:

```yaml
# order-service/application.yml
catalog:
  service:
    url: ${CATALOG_SERVICE_URL:http://localhost:8081}
```

```java
@Bean
CatalogClient catalogClient(RestClient.Builder builder,
                             @Value("${catalog.service.url}") String baseUrl) {
    RestClient restClient = builder.baseUrl(baseUrl).build();
    // ...
}
```

### Обработка ошибок

Что если catalog-service недоступен или вернул 404?

```java
public Product getProduct(Long id) {
    return catalogRestClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new ProductNotFoundException(id);
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new ServiceUnavailableException("Catalog service error");
            })
            .body(Product.class);
}
```

В следующих уроках мы обернём вызовы в Circuit Breaker — чтобы ошибки одного сервиса не роняли остальные.

### Задание

1. Создайте два Spring Boot проекта: catalog-service и order-service.
2. В catalog-service: REST API для товаров (GET /api/products, GET /api/products/{id}).
3. В order-service: при создании заказа вызывайте catalog-service для проверки товара.
4. Реализуйте вызов двумя способами: RestClient и HTTP Interface. Сравните.
5. Поднимите оба сервиса через Docker Compose. Убедитесь, что order-service вызывает catalog-service по имени контейнера.
6. Приложите скриншоты и ссылку на коммит.

---

## Урок 2. Service Discovery (Eureka)

### Зачем

В прошлом уроке мы захардкодили URL: `http://catalog-service:8081`. Проблемы:

- Если сервис переехал на другой хост/порт — надо менять конфигурацию.
- Если запущено 3 копии catalog-service — на какую отправлять запрос?
- Если одна копия упала — как узнать и не слать туда запросы?

**Service Discovery** решает все три проблемы. Сервисы **регистрируются** в реестре и **находят** друг друга по имени.

### Как это работает

```
┌──────────────┐    1. Регистрация     ┌──────────────┐
│ catalog      │ ─────────────────────▶│              │
│ service (x3) │    "я catalog-service, │   Eureka     │
└──────────────┘     мой адрес ..."    │   Server     │
                                       │              │
┌──────────────┐    2. "Где catalog?"   │  (реестр)    │
│ order        │ ─────────────────────▶│              │
│ service      │◀───────────────────── └──────────────┘
└──────────────┘    3. "Вот 3 адреса:
                     10.0.0.5:8081,
                     10.0.0.6:8081,
                     10.0.0.7:8081"

                    4. order-service сам выбирает
                       (клиентская балансировка)
```

### Когда Eureka нужна, а когда нет

| Среда | Service Discovery | Eureka нужна? |
|-------|------------------|---------------|
| Docker Compose (без K8s) | Нет встроенного | **Да** |
| Bare-metal / VM | Нет встроенного | **Да** |
| **Kubernetes** | K8s Service + DNS | **Нет** — K8s делает это сам |

> В курсе мы сначала работаем с Docker Compose → Eureka нужна. Когда перейдём к Kubernetes — Eureka можно убрать.

### Шаг 1: Eureka Server

Отдельный Spring Boot проект:

**pom.xml**:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

Не забудьте Spring Cloud BOM:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2024.0.1</version>  <!-- для Spring Boot 3.4.x -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> **Версии**: Spring Cloud `2024.0.x` совместим со Spring Boot 3.4.x. Spring Cloud `2025.0.x` — со Spring Boot 3.5.x.

**Главный класс**:

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApp.class, args);
    }
}
```

**application.yml**:

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false   # сервер сам себя не регистрирует
    fetch-registry: false          # серверу не нужен реестр от других
  server:
    enable-self-preservation: false # для разработки — отключаем защитный режим
```

Запускаем → открываем http://localhost:8761 → видим Eureka Dashboard.

### Шаг 2: Eureka Client (каждый сервис)

Добавляем в catalog-service и order-service:

**pom.xml**:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**application.yml**:

```yaml
spring:
  application:
    name: catalog-service           # ← имя, под которым сервис зарегистрируется

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true         # регистрировать IP, а не hostname
```

Запускаем сервис → в Eureka Dashboard появляется `CATALOG-SERVICE`.

### Шаг 3: Вызов по имени сервиса

Теперь order-service может вызывать catalog-service **по имени**, без URL:

```java
@Configuration
public class ClientConfig {

    @Bean
    @LoadBalanced                    // ← ключевая аннотация!
    RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }

    @Bean
    CatalogClient catalogClient(@LoadBalanced RestClient.Builder builder) {
        RestClient restClient = builder
                .baseUrl("http://catalog-service")   // ← имя из Eureka, не URL!
                .build();
        return HttpServiceProxyFactory
                .builderFor(RestClientAdapter.create(restClient))
                .build()
                .createClient(CatalogClient.class);
    }
}
```

`@LoadBalanced` подключает **Spring Cloud LoadBalancer**. Он:
1. Перехватывает запрос к `http://catalog-service`.
2. Спрашивает у Eureka: «Какие инстансы у catalog-service?»
3. Получает список адресов.
4. Выбирает один (round-robin по умолчанию).
5. Подставляет реальный адрес и выполняет запрос.

### Шаг 4: Docker Compose

```yaml
services:
  eureka:
    build: ./eureka-server
    container_name: eureka
    ports:
      - "8761:8761"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  catalog-service:
    build: ./catalog-service
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
    depends_on:
      eureka:
        condition: service_healthy

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
    depends_on:
      eureka:
        condition: service_healthy
```

Обратите внимание:
- Порт catalog-service **не пробрасывается** на хост. Он доступен только внутри Docker-сети, через Eureka.
- Переменная `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE` переопределяет `eureka.client.service-url.defaultZone`.

### Масштабирование

Запустим 3 копии catalog-service:

```bash
docker compose up -d --scale catalog-service=3
```

В Eureka Dashboard появятся 3 инстанса `CATALOG-SERVICE`. Order-service автоматически балансирует запросы между ними (round-robin).

### Eureka: под капотом

- Каждый клиент отправляет **heartbeat** каждые 30 секунд.
- Если heartbeat не приходит 90 секунд — Eureka считает инстанс мёртвым и убирает из реестра.
- Клиенты **кэшируют** реестр локально (обновляется каждые 30 сек). Если Eureka Server упал — сервисы продолжат работать по кэшу.
- **Self-preservation mode**: если Eureka теряет слишком много heartbeat'ов разом, она решает, что проблема в сети, а не в сервисах, и **не удаляет** их из реестра. В разработке это мешает — отключаем.

### Альтернативы Eureka

| Инструмент | Описание | Когда использовать |
|-----------|---------|-------------------|
| **Eureka** | Простой, встроен в Spring Cloud | Docker Compose, VM, начинающие |
| **Consul** | SD + KV-store + health checks. HashiCorp. | Нужен health checking, multi-datacenter |
| **Kubernetes DNS** | Встроен в K8s. Service → Pod'ы. | Production в Kubernetes |

### Задание

1. Создайте Eureka Server (отдельный Spring Boot проект).
2. Зарегистрируйте catalog-service и order-service в Eureka.
3. Перепишите вызов из order-service: замените URL на имя сервиса (`http://catalog-service`). Добавьте `@LoadBalanced`.
4. Поднимите всё через Docker Compose. Откройте Eureka Dashboard (http://localhost:8761) — убедитесь, что оба сервиса зарегистрированы.
5. Масштабируйте catalog-service до 3 копий (`--scale catalog-service=3`). Убедитесь, что order-service продолжает работать.
6. Приложите скриншоты и ссылку на коммит.

---

## Урок 3. API Gateway

### Зачем

У вас 5 сервисов: auth, catalog, order, notification, payment. У каждого свой порт, свой адрес. Фронтенд должен знать про все пять? А если добавится шестой?

**API Gateway** — единая точка входа для всех внешних клиентов.

```
Без Gateway:                          С Gateway:

Browser → http://auth:8081/login      Browser → http://gateway:8080/api/auth/login
Browser → http://catalog:8082/api     Browser → http://gateway:8080/api/catalog
Browser → http://order:8083/api       Browser → http://gateway:8080/api/orders
        (клиент знает 5 адресов)              (клиент знает 1 адрес)
```

### Что делает Gateway

1. **Маршрутизация** — `/api/catalog/**` → catalog-service, `/api/orders/**` → order-service.
2. **Единая аутентификация** — проверка JWT-токена в одном месте, а не в каждом сервисе.
3. **Rate Limiting** — ограничение запросов (защита от DDoS).
4. **CORS** — одна точка настройки.
5. **Логирование** — все запросы проходят через одно место.
6. **Трансформация** — добавление/удаление заголовков, переписывание путей.

### Spring Cloud Gateway

Spring Cloud Gateway — реактивный API Gateway на основе Spring WebFlux.

> С версии Spring Cloud 2025.0 также доступен **Spring Cloud Gateway Server WebMVC** — блокирующий вариант на Spring MVC. Но реактивный (WebFlux) остаётся основным и более зрелым.

**pom.xml**:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

> **Важно**: Spring Cloud Gateway работает на WebFlux. Нельзя добавлять `spring-boot-starter-web` в тот же проект — будет конфликт.

### Настройка маршрутов

**application.yml**:

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: catalog-service
          uri: lb://catalog-service          # lb:// = через LoadBalancer (Eureka)
          predicates:
            - Path=/api/catalog/**
          filters:
            - StripPrefix=1                  # убирает /api из пути

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

        - id: auth-service
          uri: lb://auth-service
          predicates:
            - Path=/api/auth/**
          filters:
            - StripPrefix=1

eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
```

Как работает:
1. Клиент отправляет `GET http://gateway:8080/api/catalog/products`.
2. Gateway видит: путь начинается с `/api/catalog/` → route `catalog-service`.
3. `StripPrefix=1` убирает первый сегмент (`/api`) → путь становится `/catalog/products`.
4. `lb://catalog-service` — Gateway спрашивает у Eureka адрес catalog-service.
5. Запрос уходит на `http://10.0.0.5:8081/catalog/products`.

### Predicates (условия маршрутизации)

Gateway выбирает маршрут по **предикатам**:

| Predicate | Описание | Пример |
|----------|---------|--------|
| `Path` | Путь | `Path=/api/orders/**` |
| `Method` | HTTP-метод | `Method=GET,POST` |
| `Header` | Заголовок | `Header=X-Request-Id, \d+` |
| `Host` | Домен | `Host=**.example.com` |
| `After` / `Before` | Время | `After=2026-01-01T00:00:00` |

### Filters (фильтры)

Фильтры модифицируют запрос или ответ:

| Filter | Описание |
|--------|---------|
| `StripPrefix=N` | Убирает N сегментов пути |
| `AddRequestHeader` | Добавляет заголовок в запрос |
| `AddResponseHeader` | Добавляет заголовок в ответ |
| `RewritePath` | Перезаписывает путь |
| `CircuitBreaker` | Интеграция с Resilience4j |
| `RequestRateLimiter` | Rate limiting |

### Глобальный фильтр для логирования

```java
@Component
@Slf4j
public class LoggingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        var request = exchange.getRequest();
        log.info("Incoming: {} {}", request.getMethod(), request.getURI());
        return chain.filter(exchange).then(Mono.fromRunnable(() ->
            log.info("Response: {} for {} {}",
                exchange.getResponse().getStatusCode(),
                request.getMethod(),
                request.getURI())
        ));
    }

    @Override
    public int getOrder() {
        return -1;  // выполнять первым
    }
}
```

### JWT-аутентификация на Gateway

Типичный паттерн: Gateway проверяет JWT-токен. Если валиден — пропускает. Если нет — 401.

```java
@Component
public class AuthFilter implements GatewayFilterFactory<AuthFilter.Config> {

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String authHeader = exchange.getRequest()
                    .getHeaders()
                    .getFirst(HttpHeaders.AUTHORIZATION);
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
            String token = authHeader.substring(7);
            // Валидация токена...
            return chain.filter(exchange);
        };
    }

    public static class Config { }
}
```

### Docker Compose

```yaml
services:
  eureka:
    build: ./eureka-server
    ports: ["8761:8761"]

  api-gateway:
    build: ./api-gateway
    ports: ["8080:8080"]                    # ← единственный публичный порт!
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
    depends_on:
      - eureka

  catalog-service:
    build: ./catalog-service
    # НЕТ ports: — доступен только через Gateway
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/

  order-service:
    build: ./order-service
    # НЕТ ports:
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
```

Наружу торчит **только Gateway** (порт 8080). Сервисы изолированы.

### Gateway в Kubernetes

В K8s роль Gateway играет **Ingress Controller** (nginx, traefik) или **Kubernetes Gateway API**. Spring Cloud Gateway можно оставить для сложной логики (JWT, rate limiting), а маршрутизацию отдать Ingress.

### Задание

1. Создайте Spring Cloud Gateway проект.
2. Настройте маршруты к catalog-service и order-service.
3. Добавьте глобальный фильтр логирования.
4. Поднимите всё через Docker Compose. Убедитесь:
   - `http://localhost:8080/api/catalog/products` → возвращает товары.
   - `http://localhost:8080/api/orders` → работает.
   - Сервисы **не** доступны напрямую (порты не проброшены).
5. В Eureka Dashboard должны быть зарегистрированы: gateway, catalog-service, order-service.
6. Приложите скриншоты и ссылку на коммит.

---

## Урок 4. Отказоустойчивость (Circuit Breaker)

### Зачем

Order-service вызывает catalog-service по REST. Catalog-service упал. Что происходит?

**Без Circuit Breaker**:
1. Order-service ждёт ответа (30 секунд таймаут по умолчанию).
2. Потоки копятся — 50 запросов × 30 секунд = 50 занятых потоков.
3. Пул потоков исчерпан → order-service перестаёт отвечать.
4. Gateway ждёт order-service → тоже зависает.
5. Пользователь видит таймаут на **все** запросы, даже те, что не связаны с каталогом.

Это **каскадный отказ**: падение одного сервиса роняет всю систему.

### Паттерн Circuit Breaker

Автоматический выключатель (как в электрике): если ток слишком большой — автомат «выбивает» цепь, защищая остальную проводку.

```
                   ┌────────────────────────┐
                   │     Circuit Breaker     │
                   │                        │
  Запросы ────────▶│  CLOSED (замкнут)      │──────▶ catalog-service
                   │  Пропускает запросы.   │
                   │  Считает ошибки.       │
                   │                        │
                   │  50% ошибок?           │
                   │         │              │
                   │         ▼              │
                   │  OPEN (разомкнут)      │──X──▶ НЕ вызывает сервис
                   │  Сразу возвращает      │       Возвращает fallback
                   │  fallback-ответ.       │
                   │  Ждёт 10 секунд.       │
                   │         │              │
                   │         ▼              │
                   │  HALF-OPEN             │──?──▶ Пробный запрос
                   │  Пропускает 1 запрос.  │
                   │  Если ОК → CLOSED      │
                   │  Если ошибка → OPEN    │
                   └────────────────────────┘
```

### Resilience4j

Spring Cloud интегрирован с **Resilience4j** — лёгкой Java-библиотекой для отказоустойчивости.

**pom.xml**:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

> `spring-boot-starter-aop` нужен для работы аннотаций `@CircuitBreaker`, `@Retry` и т.д.

### Настройка

**application.yml**:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      catalogService:                        # имя инстанса (используем в аннотации)
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10              # окно из 10 последних вызовов
        failure-rate-threshold: 50           # 50% ошибок → OPEN
        wait-duration-in-open-state: 10s     # 10 сек в OPEN, потом → HALF-OPEN
        permitted-number-of-calls-in-half-open-state: 3  # 3 пробных вызова
        minimum-number-of-calls: 5           # не открывать CB до 5 вызовов

  retry:
    instances:
      catalogService:
        max-attempts: 3                      # 3 попытки
        wait-duration: 500ms                 # пауза между попытками
        retry-exceptions:                    # на какие ошибки ретраить
          - java.net.ConnectException
          - java.net.SocketTimeoutException

  timelimiter:
    instances:
      catalogService:
        timeout-duration: 3s                 # таймаут на вызов
```

### Использование с аннотациями

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CatalogClient {

    private final RestClient catalogRestClient;

    @CircuitBreaker(name = "catalogService", fallbackMethod = "getProductFallback")
    @Retry(name = "catalogService")
    public Product getProduct(Long id) {
        log.info("Calling catalog-service for product {}", id);
        return catalogRestClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .body(Product.class);
    }

    /**
     * Fallback-метод. Вызывается когда:
     * - Circuit Breaker в состоянии OPEN
     * - Все retry-попытки исчерпаны
     * - Таймаут
     *
     * Сигнатура: те же параметры + Throwable в конце.
     */
    private Product getProductFallback(Long id, Throwable t) {
        log.warn("Catalog service unavailable, returning fallback for product {}", id, t);
        return new Product(id, "Unknown Product", BigDecimal.ZERO, "Catalog service is temporarily unavailable");
    }
}
```

### Как это работает (сценарий)

```
1. catalog-service работает:
   Запрос → CB (CLOSED) → catalog → ответ ✅

2. catalog-service падает:
   Запрос 1 → CB (CLOSED) → catalog → ошибка ❌ (1/10)
   Запрос 2 → CB (CLOSED) → catalog → ошибка ❌ (2/10)
   ...
   Запрос 5 → CB (CLOSED) → catalog → ошибка ❌ (5/10 = 50%)
   → CB переходит в OPEN

3. catalog-service всё ещё лежит:
   Запрос 6 → CB (OPEN) → сразу fallback ⚡ (не ждём 30 секунд!)
   Запрос 7 → CB (OPEN) → сразу fallback ⚡
   → Потоки не копятся, order-service продолжает работать

4. Прошло 10 секунд:
   Запрос 8 → CB (HALF-OPEN) → catalog → ошибка ❌
   → CB снова OPEN

5. catalog-service починили:
   Через 10 сек: Запрос → CB (HALF-OPEN) → catalog → ответ ✅
   → CB переходит в CLOSED, нормальная работа
```

### Паттерны Resilience4j

| Паттерн | Зачем | Пример |
|---------|-------|--------|
| **Circuit Breaker** | Прекращает вызовы к мёртвому сервису | Catalog упал → сразу fallback |
| **Retry** | Повторяет при временных ошибках | Сеть мигнула → повторим 3 раза |
| **Time Limiter** | Таймаут на вызов | Не ждём дольше 3 секунд |
| **Bulkhead** | Ограничивает параллельные вызовы | Макс 10 одновременных к catalog |
| **Rate Limiter** | Ограничивает частоту вызовов | Макс 100 запросов/секунду |

### Порядок выполнения

Если навешено несколько аннотаций, порядок важен:

```
Запрос → Retry → CircuitBreaker → TimeLimiter → Вызов
```

Retry оборачивает CB. Это значит: каждая retry-попытка считается отдельным вызовом для CB.

### Мониторинг Circuit Breaker

Добавьте Actuator:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

Теперь по адресу `/actuator/circuitbreakers` можно увидеть состояние всех CB:

```bash
curl http://localhost:8082/actuator/circuitbreakers
# {"circuitBreakers":{"catalogService":{"state":"CLOSED","failureRate":-1.0}}}
```

### Circuit Breaker на Gateway

Можно добавить CB прямо на уровне Gateway:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: catalog-service
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - name: CircuitBreaker
              args:
                name: catalogServiceCB
                fallbackUri: forward:/fallback/catalog
```

```java
@RestController
public class FallbackController {
    @GetMapping("/fallback/catalog")
    public Map<String, String> catalogFallback() {
        return Map.of("status", "Catalog service is temporarily unavailable");
    }
}
```

### Задание

1. Добавьте Resilience4j в order-service.
2. Оберните вызов catalog-service в `@CircuitBreaker` + `@Retry` с fallback-методом.
3. Поднимите систему через Docker Compose.
4. Проверьте:
   - Нормальная работа (оба сервиса UP).
   - Остановите catalog-service (`docker compose stop catalog-service`). Убедитесь, что order-service продолжает отвечать (fallback-ответ).
   - Поднимите catalog-service обратно. Убедитесь, что order-service вернулся к нормальной работе.
5. Посмотрите состояние CB через `/actuator/circuitbreakers`.
6. Приложите скриншоты и ссылку на коммит.
