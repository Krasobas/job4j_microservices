# Микросервисы: пошаговый план создания

> Шпаргалка. Что, когда, зачем, в каком порядке.
>
> Стек: Java 21, Spring Boot 3.4, Spring Cloud 2025.0.x

---

## Карта маршрута

```
Шаг 0   Пойми, нужны ли тебе микросервисы вообще
  │
Шаг 1   Разбей домен на сервисы
  │
Шаг 2   Каждый сервис = отдельный Spring Boot проект + своя БД
  │
Шаг 3   Синхронное общение: REST (RestClient / HTTP Interface)
  │
Шаг 4   Service Discovery: Eureka
  │
Шаг 5   API Gateway
  │
Шаг 6   Централизованная конфигурация: Config Server
  │
Шаг 7   Отказоустойчивость: Circuit Breaker (Resilience4j)
  │
Шаг 8   Асинхронное общение: Kafka
  │
Шаг 9   Распределённые транзакции: паттерн Saga
  │
Шаг 10  Наблюдаемость: логи, метрики, трейсинг
  │
Шаг 11  Docker + Docker Compose (локальная сборка)
  │
Шаг 12  Kubernetes (деплой в кластер)
```

---

## Шаг 0. А нужны ли микросервисы?

**Не начинай с микросервисов.** Начни с монолита.

Микросервисы — это **не цель**, а инструмент решения конкретных проблем. Они добавляют сложность: сеть, распределённые транзакции, деплой, мониторинг. Если у тебя команда из 2–3 человек и один продукт — монолит будет быстрее, проще, дешевле.

**Переходи на микросервисы, когда:**

- Монолит стал слишком большим — сборка 20+ минут, деплой страшно делать.
- Разные части системы масштабируются по-разному (каталог нагружен, а биллинг нет).
- Над разными частями работают разные команды и мешают друг другу.
- Нужна технологическая гибкость (часть на Java, часть на Python/Go).

**Не переходи, если:**

- «Все так делают» — это не причина.
- Команда маленькая (1–5 человек).
- Проект на старте и доменная модель ещё не устоялась.

> **Правило**: если не можешь нарисовать границы сервисов за 15 минут — значит, ты ещё не понимаешь домен достаточно хорошо. Оставайся в монолите, пока границы не станут очевидными.

---

## Шаг 1. Разбей домен на сервисы

### Как резать

Сервис = **бизнес-возможность** (business capability), а не техническая прослойка.

```
✅ Правильно (по бизнес-доменам):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐
│  Auth    │ │  Orders  │ │ Catalog  │ │Notification│
│ Service  │ │ Service  │ │ Service  │ │  Service   │
└──────────┘ └──────────┘ └──────────┘ └───────────┘

❌ Неправильно (по техническим слоям):
┌──────────┐ ┌──────────┐ ┌──────────┐
│Controller│ │ Service  │ │   DAO    │
│ Service  │ │ Service  │ │ Service  │
└──────────┘ └──────────┘ └──────────┘
```

### Принципы

- **Один сервис — одна ответственность.** Order Service знает всё о заказах и ничего о каталоге товаров.
- **Своя база данных у каждого сервиса.** Никаких shared database. Order Service не лезет в таблицы Catalog Service.
- **Loose coupling, high cohesion.** Сервисы слабо связаны друг с другом, но внутри — сильно связная логика.
- **Если два сервиса всегда деплоятся вместе** — это один сервис, разрезанный зря.

### Bounded Context (DDD)

Если слышал про Domain-Driven Design — каждый сервис это **Bounded Context**. У каждого контекста своя модель. `User` в Auth Service — это логин и пароль. `User` в Order Service — это customerId и адрес доставки. Это разные объекты, даже если кажутся одним и тем же.

---

## Шаг 2. Создай сервисы

Каждый сервис — отдельный Spring Boot проект со своим:
- `pom.xml` / `build.gradle`
- `application.yml`
- Базой данных
- Git-репозиторием (или модулем в multi-module проекте)

```
project-root/
├── auth-service/          ← Spring Boot app, порт 8081
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── order-service/         ← Spring Boot app, порт 8082
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── catalog-service/       ← Spring Boot app, порт 8083
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── notification-service/  ← Spring Boot app, порт 8084
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
└── compose.yaml           ← всё поднимается локально
```

### Порты

На этапе разработки каждый сервис слушает свой порт. В Kubernetes это неважно — каждый Pod имеет свой IP.

### БД на сервис

```yaml
# order-service/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    
# catalog-service/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/catalog
```

Разные базы (или хотя бы разные схемы). Это жёсткое правило.

---

## Шаг 3. Синхронное общение (REST)

Сервисы должны общаться. Самый простой способ — HTTP REST.

### Когда REST

- Нужен **ответ прямо сейчас** (запрос-ответ).
- Операции чтения (GET) — почти всегда REST.
- Простые команды, где важен результат.

### Чем вызывать

| Инструмент | Статус | Когда использовать |
|-----------|--------|-------------------|
| `RestTemplate` | **Deprecated** (Spring 5.0+) | Не используй |
| `WebClient` | Актуален | Реактивные приложения (WebFlux) |
| **`RestClient`** | ✅ Рекомендуется (Spring 6.1+) | Блокирующие приложения (Spring MVC) |
| **HTTP Interface** | ✅ Рекомендуется (Spring 6+) | Декларативный стиль (как Feign, но встроен) |
| OpenFeign | Работает, но уступает HTTP Interface | Legacy-проекты |

### RestClient (императивный стиль)

```java
@Bean
RestClient orderRestClient() {
    return RestClient.builder()
        .baseUrl("http://order-service:8082")
        .build();
}

// Использование
Order order = restClient.get()
    .uri("/api/orders/{id}", orderId)
    .retrieve()
    .body(Order.class);
```

### HTTP Interface (декларативный стиль)

```java
// Объявляем интерфейс
public interface OrderClient {
    @GetExchange("/api/orders/{id}")
    Order getOrder(@PathVariable Long id);

    @PostExchange("/api/orders")
    Order createOrder(@RequestBody OrderRequest request);
}

// Создаём бин
@Bean
OrderClient orderClient(RestClient.Builder builder) {
    RestClient restClient = builder.baseUrl("http://order-service:8082").build();
    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(restClient))
        .build()
        .createClient(OrderClient.class);
}
```

> HTTP Interface — замена OpenFeign. Встроен в Spring, не нужна отдельная зависимость.

### Проблема: захардкоженные URL

Пока мы пишем `http://order-service:8082` руками. Если сервис переехал на другой хост или порт — всё сломается. Решение — Service Discovery (следующий шаг).

---

## Шаг 4. Service Discovery (Eureka)

### Зачем

Сервисы не должны знать IP-адреса друг друга. Они должны **регистрироваться** в реестре и **находить** друг друга по имени.

```
┌────────────┐   регистрация    ┌──────────┐
│ Order      │ ───────────────▶ │  Eureka  │
│ Service    │                  │  Server  │
└────────────┘                  └──────────┘
                                     ▲
┌────────────┐   "где Order?"   │
│ Catalog    │ ─────────────────┘
│ Service    │ ◀── "вот: 10.0.0.5:8082"
└────────────┘
```

### Когда подключать

**Сразу после того, как у тебя появился второй сервис**, который вызывает первый. Если сервисы общаются — им нужен Service Discovery.

### Eureka Server

Отдельный Spring Boot проект:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp { ... }
```

```yaml
# eureka-server/application.yml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false     # сам себя не регистрирует
    fetch-registry: false
```

### Eureka Client (каждый сервис)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
# order-service/application.yml
spring:
  application:
    name: order-service              # имя, по которому найдут этот сервис
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Вызов по имени сервиса

Теперь вместо `http://localhost:8082` пишем `http://order-service`:

```java
@Bean
@LoadBalanced                       // ← включает клиентскую балансировку
RestClient.Builder restClientBuilder() {
    return RestClient.builder();
}
```

Spring Cloud LoadBalancer перехватит запрос, спросит у Eureka «где order-service?», получит список адресов и распределит нагрузку.

### Альтернативы Eureka

| Инструмент | Описание |
|-----------|---------|
| **Eureka** | Стандарт Spring Cloud. Простой, проверенный. |
| **Consul** | Больше фич (KV-store, health checks). Отдельный процесс. |
| **Kubernetes DNS** | Если деплой в K8s — Service Discovery встроен. Eureka не нужна. |

> В Kubernetes сервисы находят друг друга через K8s Service + DNS. Eureka нужна, если деплоишь **без Kubernetes**.

---

## Шаг 5. API Gateway

### Зачем

Клиент (фронтенд, мобилка) не должен знать о внутренних сервисах. Ему нужна **одна точка входа**.

```
                    ┌──────────────┐
  Client ──────────▶│  API Gateway │
                    │  (порт 8080) │
                    └──────┬───────┘
                           │ маршрутизация по path
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         /api/auth    /api/orders  /api/catalog
        ┌────────┐   ┌────────┐   ┌────────┐
        │  Auth  │   │ Orders │   │Catalog │
        │ :8081  │   │ :8082  │   │ :8083  │
        └────────┘   └────────┘   └────────┘
```

### Что делает Gateway

- **Маршрутизация** — `/api/orders/**` → Order Service.
- **Аутентификация/авторизация** — проверка JWT до того, как запрос дойдёт до сервиса.
- **Rate limiting** — ограничение количества запросов.
- **CORS** — единая точка настройки.
- **Логирование** — все запросы проходят через одно место.

### Когда подключать

**Когда появляется внешний клиент** (фронтенд, мобильное приложение). Если сервисы общаются только между собой — Gateway не нужен.

### Spring Cloud Gateway

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
# gateway/application.yml
server:
  port: 8080
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: order-service
              uri: lb://order-service        # lb:// = LoadBalanced через Eureka
              predicates:
                - Path=/api/orders/**
            - id: catalog-service
              uri: lb://catalog-service
              predicates:
                - Path=/api/catalog/**
```

`lb://order-service` — Gateway спрашивает у Eureka адрес и балансирует.

### Альтернатива: Spring Cloud Gateway MVC

Начиная с Spring Cloud 2025.0, доступна версия Gateway на Spring MVC (не WebFlux). Удобнее, если весь проект блокирующий:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webmvc</artifactId>
</dependency>
```

---

## Шаг 6. Централизованная конфигурация

### Зачем

У тебя 10 сервисов и у каждого `application.yml`. Поменять URL базы данных — нужно лезть в 10 мест. Config Server решает это: конфигурация хранится **в одном месте** (Git-репозиторий), а сервисы забирают её при старте.

### Когда подключать

**Когда сервисов стало больше 3–4** и ты замечаешь, что дублируешь конфигурацию. На ранних этапах — не нужен, усложняет без пользы.

### Config Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```yaml
# config-server/application.yml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
```

В Git-репозитории:

```
config-repo/
├── order-service.yml
├── catalog-service.yml
├── application.yml          ← общая для всех
└── order-service-prod.yml   ← профиль prod для order-service
```

### Config Client (каждый сервис)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```yaml
# order-service/application.yml
spring:
  config:
    import: optional:configserver:http://localhost:8888
```

### Альтернатива в Kubernetes

Если деплоишь в K8s — ConfigMap и Secret заменяют Config Server. Можно обойтись без него.

---

## Шаг 7. Отказоустойчивость (Circuit Breaker)

### Зачем

Сервис A вызывает сервис B. Сервис B лёг. Без защиты сервис A будет ждать ответа, копить потоки и тоже ляжет. **Каскадный отказ** — когда падение одного сервиса роняет всю систему.

### Паттерн Circuit Breaker

```
        Запросы к сервису B
              │
    ┌─────────┴─────────┐
    │   Circuit Breaker  │
    │                    │
    │  CLOSED ──────────── нормальная работа, пропускает запросы
    │    │                │
    │    │ N ошибок подряд │
    │    ▼                │
    │  OPEN ───────────── блокирует запросы, сразу возвращает fallback
    │    │                │
    │    │ таймаут         │
    │    ▼                │
    │  HALF-OPEN ─────── пропускает пробный запрос
    │    │                │
    │    ├─ успех → CLOSED │
    │    └─ ошибка → OPEN  │
    └─────────────────────┘
```

### Когда подключать

**Когда сервисы начинают зависеть друг от друга по сети.** Если Order Service вызывает Catalog Service — оберни вызов в Circuit Breaker. Чем раньше, тем лучше.

### Resilience4j

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      catalogService:
        sliding-window-size: 10
        failure-rate-threshold: 50     # 50% ошибок → OPEN
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      catalogService:
        max-attempts: 3
        wait-duration: 500ms
  timelimiter:
    instances:
      catalogService:
        timeout-duration: 3s
```

```java
@CircuitBreaker(name = "catalogService", fallbackMethod = "fallback")
@Retry(name = "catalogService")
public Product getProduct(Long id) {
    return catalogClient.getProduct(id);
}

public Product fallback(Long id, Throwable t) {
    return Product.unknown(id);  // возвращаем заглушку вместо ошибки
}
```

### Дополнительные паттерны Resilience4j

| Паттерн | Зачем |
|---------|-------|
| **Circuit Breaker** | Прекращает вызовы к мёртвому сервису |
| **Retry** | Повторяет при временных ошибках |
| **Rate Limiter** | Ограничивает частоту вызовов |
| **Bulkhead** | Изолирует потоки (один тормозящий сервис не съедает все потоки) |
| **Time Limiter** | Таймаут на вызов |

---

## Шаг 8. Асинхронное общение (Kafka)

### Зачем

REST — синхронный: вызвал → ждёшь → получил ответ. Это создаёт **временнýю связность** — если получатель недоступен, отправитель блокируется.

Kafka (и брокеры сообщений в целом) убирают эту связность. Отправитель кладёт сообщение в топик и идёт дальше. Получатель прочитает когда сможет.

```
┌────────┐   publish    ┌───────────┐   consume    ┌─────────────┐
│ Order  │ ───────────▶ │   Kafka   │ ───────────▶ │Notification │
│Service │              │  (topic)  │              │  Service    │
└────────┘              └───────────┘              └─────────────┘
     │                       ▲
     │ "заказ создан"        │
     │ и пошёл дальше        │ сообщение хранится,
     │ не ждёт ответа        │ даже если consumer упал
```

### Когда REST, когда Kafka

| Критерий | REST | Kafka |
|---------|------|-------|
| Нужен ответ? | Да | Нет |
| Получатель может быть недоступен? | Нет (ошибка) | Да (прочитает позже) |
| Одному отправителю — много получателей? | Неудобно | Идеально (fan-out) |
| Важен порядок? | Нет гарантий | Гарантирован (в рамках partition) |
| Пример | GET /orders/{id} | «Заказ создан → уведомление + аналитика + склад» |

**Правило**: GET-запросы → REST. «Событие произошло» → Kafka. POST, где важен только факт доставки, но не результат → часто Kafka.

### Когда подключать

**Когда появляются события**, которые должны обрабатываться другими сервисами без ожидания ответа. Типичные сценарии:
- Уведомления (email, push, Telegram)
- Аналитика и логирование
- Обновление поисковых индексов
- Интеграция со сторонними системами

### Spring + Kafka

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

**Producer (Order Service):**

```java
@RequiredArgsConstructor
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        var event = new OrderEvent("ORDER_CREATED", order.getId(), order);
        kafkaTemplate.send("order-events", order.getId().toString(), event);
    }
}
```

**Consumer (Notification Service):**

```java
@Component
public class OrderEventConsumer {
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderEvent(OrderEvent event) {
        if ("ORDER_CREATED".equals(event.type())) {
            sendNotification(event);
        }
    }
}
```

### Kafka: ключевые концепции

```
                    ┌─── Topic: order-events ───┐
                    │                           │
  Producer ────────▶│  Partition 0: [m1][m2][m3]│───────▶ Consumer Group A
                    │  Partition 1: [m4][m5]    │              (notification)
                    │  Partition 2: [m6][m7][m8]│───────▶ Consumer Group B
                    │                           │              (analytics)
                    └───────────────────────────┘
```

- **Topic** — именованный канал сообщений.
- **Partition** — параллельная единица внутри топика. Порядок гарантирован **внутри partition**.
- **Consumer Group** — группа потребителей. Каждое сообщение читает **ровно один** consumer в группе.
- **Offset** — позиция чтения. Kafka хранит сообщения на диске (по умолчанию 7 дней).

### Docker Compose для Kafka (KRaft, без Zookeeper)

```yaml
services:
  kafka:
    image: apache/kafka:3.9.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

> **KRaft mode** — с Kafka 3.5+ Zookeeper не нужен. Kafka управляет метаданными самостоятельно.

---

## Шаг 9. Распределённые транзакции (Saga)

### Проблема

В монолите: «создай заказ + спиши деньги + зарезервируй товар» — одна транзакция БД. Либо всё, либо ничего.

В микросервисах: каждый шаг — вызов **отдельного** сервиса со **своей** БД. Общей транзакции нет. Если товар зарезервировали, а деньги не списались — нужно откатить резервацию.

### Паттерн Saga

Saga — последовательность локальных транзакций, где каждый шаг имеет **компенсирующее действие** (откат).

```
Создать заказ ──▶ Списать деньги ──▶ Зарезервировать товар ──▶ ✅ Успех
       │                │                     │
  (компенсация)    (компенсация)         (компенсация)
  Отменить заказ   Вернуть деньги    Снять резерв
       │                │                     │
       ◀────────────────◀─────────────────────◀── ❌ Ошибка на любом шаге
```

### Два подхода

**Хореография (Choreography)** — каждый сервис слушает события и решает сам, что делать:

```
Order Service ──"order.created"──▶ Kafka ──▶ Payment Service
                                                    │
Payment Service ──"payment.completed"──▶ Kafka ──▶ Inventory Service
                                                    │
Inventory Service ──"inventory.reserved"──▶ Kafka ──▶ Order Service (confirm)
```

Плюсы: нет единой точки отказа, сервисы независимы.
Минусы: сложно отследить весь поток, сложнее дебажить.

**Оркестрация (Orchestration)** — один сервис (оркестратор) координирует все шаги:

```
                 ┌──────────────────┐
                 │   Order Saga     │
                 │   Orchestrator   │
                 └───────┬──────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌────────────┐ ┌──────────┐ ┌───────────┐
    │  Payment   │ │Inventory │ │  Order    │
    │  Service   │ │ Service  │ │  Service  │
    └────────────┘ └──────────┘ └───────────┘
```

Плюсы: весь поток виден в одном месте, легче дебажить.
Минусы: оркестратор — единая точка отказа.

### Когда подключать

**Когда появляются операции, затрагивающие несколько сервисов**, и нужна согласованность данных. Обычно это: оформление заказа, денежные переводы, бронирования.

### Что выбрать

- **2–3 шага** → хореография (проще).
- **4+ шагов** или **сложная логика откатов** → оркестрация (контролируемее).

---

## Шаг 10. Наблюдаемость (Observability)

### Три столпа

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│    Логи     │  │   Метрики   │  │  Трейсинг   │
│ (что было)  │  │ (как дела)  │  │ (где был    │
│             │  │             │  │  запрос)    │
│ Loki/ELK    │  │ Prometheus  │  │ Zipkin/     │
│             │  │ + Grafana   │  │ Jaeger      │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Когда подключать

**С самого начала.** Серьёзно. Без логов и метрик ты слеп. Без трейсинга не поймёшь, какой из 10 сервисов тормозит.

### Логирование

Централизованные логи — все сервисы пишут в одно место.

Простейший вариант: JSON-логи → Loki → Grafana.

```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

Или ELK-стек (Elasticsearch + Logstash + Kibana) — мощнее, но тяжелее.

### Метрики (Prometheus + Grafana)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

Prometheus скрейпит `/actuator/prometheus` → Grafana визуализирует.

### Distributed Tracing (Micrometer Tracing)

Запрос проходит через 5 сервисов. Как понять, где он тормозит?

**Trace** — весь путь запроса. **Span** — отрезок пути внутри одного сервиса.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0    # 100% запросов трейсить (в prod обычно 0.1)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

> **Sleuth deprecated.** Начиная с Spring Boot 3, вместо Spring Cloud Sleuth используется **Micrometer Tracing**. Sleuth не обновляется.

---

## Шаг 11. Docker + Docker Compose

### Зачем

На этом этапе у тебя 5–10 сервисов + Kafka + PostgreSQL + Eureka + Gateway. Запускать их вручную нереально. Docker Compose поднимает всё одной командой.

### Когда подключать

**Как только появился второй сервис.** Docker Compose — must-have для локальной разработки.

### Типовой compose.yaml

```yaml
services:
  # ── Инфраструктура ─────────────────────
  eureka:
    build: ./eureka-server
    ports: ["8761:8761"]

  config-server:
    build: ./config-server
    ports: ["8888:8888"]
    depends_on:
      eureka:
        condition: service_healthy

  gateway:
    build: ./api-gateway
    ports: ["8080:8080"]
    depends_on:
      - eureka

  kafka:
    image: apache/kafka:3.9.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  # ── Базы данных ────────────────────────
  orders-db:
    image: postgres:17
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - orders-data:/var/lib/postgresql/data

  catalog-db:
    image: postgres:17
    environment:
      POSTGRES_DB: catalog
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - catalog-data:/var/lib/postgresql/data

  # ── Сервисы ────────────────────────────
  order-service:
    build: ./order-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://orders-db:5432/orders
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      - orders-db
      - eureka
      - kafka

  catalog-service:
    build: ./catalog-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://catalog-db:5432/catalog
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
    depends_on:
      - catalog-db
      - eureka

volumes:
  orders-data:
  catalog-data:
```

```bash
docker compose up -d
docker compose logs -f order-service
```

---

## Шаг 12. Kubernetes

### Зачем

Docker Compose — для одной машины. Kubernetes — для production: масштабирование, отказоустойчивость, zero-downtime деплой.

### Когда подключать

**Когда готов к production-деплою.** Для разработки Compose хватает. K8s — это следующий уровень.

### Ключевое отличие: Eureka не нужна

В Kubernetes Service Discovery **встроен**. Kubernetes Service + DNS заменяют Eureka. Адрес сервиса — это имя K8s Service:

```yaml
SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-svc:5432/tracker
```

Можно оставить Eureka (Spring Cloud Kubernetes умеет), но в большинстве случаев проще использовать нативный K8s DNS.

### Что меняется при переходе на K8s

| Docker Compose | Kubernetes |
|---------------|-----------|
| `compose.yaml` | Deployment + Service YAML |
| `environment` | ConfigMap + Secret |
| `volumes` | PersistentVolumeClaim |
| `depends_on` | readinessProbe + initContainers |
| Eureka | K8s Service + DNS |
| Один nginx | Ingress Controller |
| `docker compose up` | `kubectl apply -f k8s/` |

---

## Шпаргалка: когда что подключать

```
Количество сервисов:    1         2-3         4-7         8+
                        │          │            │           │
REST-клиент             │    RestClient /  HTTP Interface   │
                        │          │            │           │
Service Discovery       │       Eureka ─────────────────────│
                        │          │            │           │
Circuit Breaker         │     Resilience4j ─────────────────│
                        │          │            │           │
API Gateway             │          │      Spring Cloud      │
                        │          │      Gateway           │
Config Server           │          │            │     Config Server
                        │          │            │           │
Kafka                   │          │    по потребности ──────│
                        │          │            │           │
Docker Compose          │    сразу ─────────────────────────│
                        │          │            │           │
Observability           │    сразу ─────────────────────────│
                        │          │            │           │
Kubernetes              │          │            │    production
                        │          │            │           │
Saga                    │          │     когда транзакция   │
                        │          │     затрагивает 2+     │
                        │          │     сервиса            │
```

---

## Антипаттерны (чего НЕ делать)

1. **Shared database** — два сервиса пишут в одну таблицу. Потеряна вся независимость.

2. **Synchronous chain** — A вызывает B, B вызывает C, C вызывает D синхронно. Любой упавший ломает всю цепочку. Используй Kafka для длинных цепочек.

3. **Distributed monolith** — 10 сервисов, но деплоятся все вместе, потому что все друг от друга зависят. Это монолит, только хуже.

4. **Nano-сервисы** — сервис из одного CRUD-контроллера. Overhead на инфраструктуру больше, чем польза. Если сомневаешься — объединяй.

5. **REST для событий** — Order Service вызывает POST на Notification Service. А если Notification Service упал? Используй Kafka для fire-and-forget.

6. **Одна библиотека моделей на все сервисы** — shared-lib с Entity/DTO. Обновил — передеплой всех. Дублирование DTO между сервисами — это нормально.

7. **Игнорирование idempotency** — Kafka может доставить сообщение дважды. REST может быть вызван повторно (retry). Каждый обработчик должен быть **идемпотентным** (повторный вызов не создаёт дубль).

---

## Порядок запуска инфраструктуры

Когда запускаешь всё локально — порядок важен:

```
1. БД (PostgreSQL) ─── данные должны быть готовы
   │
2. Kafka ───────────── брокер должен быть доступен
   │
3. Eureka Server ───── реестр должен ждать регистрации
   │
4. Config Server ───── (если есть) конфиги раздаются отсюда
   │
5. Бизнес-сервисы ──── регистрируются в Eureka, подключаются к БД и Kafka
   │
6. API Gateway ─────── маршрутизирует к сервисам (ждёт, пока они зарегистрируются)
```

В `compose.yaml` это обеспечивается через `depends_on` + `healthcheck`.

---

## Чек-лист готовности к production

- [ ] Каждый сервис имеет свою БД
- [ ] REST-вызовы обёрнуты в Circuit Breaker
- [ ] События идут через Kafka (не REST)
- [ ] Все сервисы регистрируются в Service Discovery
- [ ] Один вход для клиента (API Gateway)
- [ ] Health checks настроены (`/actuator/health`)
- [ ] Логи централизованы (JSON, Loki/ELK)
- [ ] Метрики экспортируются (Prometheus)
- [ ] Трейсинг работает (Micrometer → Zipkin/Jaeger)
- [ ] Dockerfile использует multi-stage build
- [ ] Docker Compose поднимает всё локально
- [ ] Kubernetes-манифесты готовы для production
- [ ] Обработчики идемпотентны
- [ ] Секреты не в коде (Secret / Vault / env-файлы)
- [ ] README описывает, как запустить проект
