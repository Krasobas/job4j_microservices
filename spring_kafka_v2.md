# 4. Spring Boot + Kafka [v2, 2026]

## Введение

В предыдущем уроке мы развернули Kafka через Docker и поработали с ней через консольные утилиты. Теперь подключим Kafka к Spring Boot приложению.

> **Что изменилось по сравнению с оригинальным уроком:**
> - Spring Boot 3.4 вместо 2.x
> - Конфигурация через `application.yml` с type-safe binding вместо длинных строк в `.properties`
> - `JsonDeserializer` настроен через Spring Boot properties (без ручного указания полных имён классов)
> - Показана типизированная `KafkaTemplate<String, Order>` вместо `KafkaTemplate<String, Object>`

## Сценарий

Представьте, что мы разрабатываем сервис доставки еды.

```
┌────────────┐     Kafka      ┌──────────────┐
│   Order    │───────────────▶│   Kitchen    │
│  Service   │  topic:        │   Service    │
│ (producer) │  "job4j_orders"│  (consumer)  │
└────────────┘                └──────────────┘
```

Пользователь делает заказ через сервис Order. Order сохраняет заказ в свою БД и отправляет сообщение в Kafka. Сервис Kitchen слушает тему `job4j_orders` и получает заказ для приготовления.

## Зависимость

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

Одна зависимость — Spring Boot автоконфигурация сделает остальное. Версия подтягивается из `spring-boot-starter-parent`.

## Конфигурация

### application.yml (сервис Order — producer)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### application.yml (сервис Kitchen — consumer)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: kitchen-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
```

Разберём ключевые настройки:

- `bootstrap-servers: localhost:9092` — адрес Kafka (из нашего Docker-compose).
- `key-serializer` / `value-serializer` — как превращать объекты в байты для отправки. Ключ — строка, значение — JSON (через Jackson).
- `key-deserializer` / `value-deserializer` — обратная операция при чтении.
- `group-id: kitchen-group` — группа потребителей. Все инстансы Kitchen Service в одной группе — Kafka распределит секции между ними.
- `auto-offset-reset: earliest` — при первом запуске читать с начала темы (а не только новые сообщения).
- `spring.json.trusted.packages: "*"` — разрешить десериализацию любых классов. В продакшене лучше указать конкретные пакеты: `ru.job4j.order.model`.

## Код

### Модель (общая для обоих сервисов)

```java
package ru.job4j.order.model;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class Order {
    private Long id;
    private Long dishId;
}
```

### OrderService (producer)

```java
package ru.job4j.order.service;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import ru.job4j.order.model.Order;
import ru.job4j.order.repository.OrderRepository;

@Slf4j
@Service
@AllArgsConstructor
public class OrderService {

    private final KafkaTemplate<String, Order> kafkaTemplate;
    private final OrderRepository orderRepository;

    public Order save(Order order) {
        var savedOrder = orderRepository.save(order);
        kafkaTemplate.send("job4j_orders", savedOrder);
        log.info("Заказ отправлен в Kafka: {}", savedOrder);
        return savedOrder;
    }
}
```

Что здесь происходит:
1. Заказ сохраняется в БД сервиса Order (локальная транзакция).
2. `kafkaTemplate.send("job4j_orders", savedOrder)` — отправляет объект `Order` в тему `job4j_orders`. Jackson сериализует его в JSON автоматически.
3. Первый параметр — имя темы, второй — объект-сообщение. Kafka создаст тему автоматически (если включена автосоздание), но лучше создать заранее.

> **Типизация:** в оригинале использовался `KafkaTemplate<String, Object>`. Мы используем `KafkaTemplate<String, Order>` — это безопаснее, ловит ошибки на этапе компиляции.

### KitchenService (consumer)

```java
package ru.job4j.kitchen.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import ru.job4j.order.model.Order;

@Service
@Slf4j
public class KitchenService {

    @KafkaListener(topics = "job4j_orders", groupId = "kitchen-group")
    public void receiveOrder(Order order) {
        log.info("Получен заказ из Kafka: {}", order);
        /* здесь начинается приготовление */
    }
}
```

`@KafkaListener` — ключевая аннотация. Spring Kafka автоматически:
1. Создаёт consumer, подключает его к Kafka.
2. Подписывает на тему `job4j_orders`.
3. При получении сообщения — десериализует JSON в `Order` и вызывает метод `receiveOrder`.
4. Управляет offset'ами — после успешной обработки сдвигает позицию, чтобы не читать одно сообщение дважды.

## Проверка

1. Убедитесь, что Kafka запущена (`docker compose up -d`).
2. Запустите сервис Kitchen (consumer).
3. Запустите сервис Order (producer).
4. Отправьте POST через Postman:

```
POST http://localhost:8080/api/orders
Content-Type: application/json

{
  "dishId": 2
}
```

5. В логах Kitchen увидите:

```
Получен заказ из Kafka: Order(id=1, dishId=2)
```

## Что происходит под капотом

```
Postman                    Order Service                Kafka              Kitchen Service
  │                             │                         │                      │
  │── POST /api/orders ────────▶│                         │                      │
  │                             │                         │                      │
  │                             │── save to DB            │                      │
  │                             │                         │                      │
  │                             │── send("job4j_orders",  │                      │
  │                             │    Order{id=1,dishId=2})│                      │
  │                             │────────────────────────▶│                      │
  │                             │                         │                      │
  │◀── 200 OK + Order ─────────│                         │                      │
  │                             │                         │                      │
  │                             │                         │── Order{id=1,...} ──▶│
  │                             │                         │  (асинхронно!)       │
  │                             │                         │                      │
  │                             │                         │            receiveOrder()
  │                             │                         │            log: "Получен..."
```

Обратите внимание: Order Service вернул `200 OK` клиенту **до того**, как Kitchen получил сообщение. Это и есть асинхронность — отправитель не ждёт получателя.

## Ссылки

- [Spring for Apache Kafka — Reference](https://docs.spring.io/spring-kafka/reference/)
- [KafkaTemplate — API](https://docs.spring.io/spring-kafka/api/org/springframework/kafka/core/KafkaTemplate.html)
- [Apache Kafka — Introduction](https://kafka.apache.org/intro)

## Задание

1. Откройте проект CheckDev. Сервис notification принимает синхронные сообщения через REST API. Но в этом нет необходимости, потому что все события здесь асинхронные. Ваша задача — переписать сервис notification на использование Kafka.
2. Создайте отдельную ветку для выполнения этой задачи. Оставьте ссылку на коммит.
3. Переведите ответственного на Петра Арсентьева.
