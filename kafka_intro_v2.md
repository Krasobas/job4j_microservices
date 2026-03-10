# 3. Kafka. Введение. [v2, 2026]

## Что такое брокер сообщений

В микросервисной архитектуре сервисы общаются друг с другом. Мы уже видели **синхронное** взаимодействие (RestClient, WebClient). Но есть сценарии, где синхронность не подходит: сервис отправил сообщение и не хочет ждать ответа. Для этого используются **брокеры сообщений**.

Брокер сообщений — посредник между сервисами. Аналогия — почтовое отделение: вы отправляете письмо, почта его доставляет, а получатель забирает, когда ему удобно. Отправителю не нужно знать, где находится получатель и онлайн ли он.

```
┌──────────┐          ┌────────────┐          ┌──────────┐
│ Сервис A │──msg───▶ │   Брокер   │──msg───▶ │ Сервис B │
│ (sender) │          │ (Kafka)    │          │(receiver)│
└──────────┘          └────────────┘          └──────────┘
                           │
                           │──msg───▶ ┌──────────┐
                           │          │ Сервис C │
                           │          │(receiver)│
                           │          └──────────┘
```

Ключевые понятия:
- **Сообщение (Message)** — данные + метаданные (тема, ключ, заголовки).
- **Тема (Topic)** — именованный канал для сообщений. Получатели подписываются на темы. Аналогия — рассылка: подписался на «заказы» — получаешь заказы.
- **Отправитель (Producer)** — сервис, который публикует сообщения в тему.
- **Получатель (Consumer)** — сервис, который читает сообщения из темы.
- **Асинхронность** — отправитель не ждёт ответа. Когда сообщение придёт — получатель его обработает. За это отвечают **слушатели** (listeners).

## Apache Kafka

Kafka — один из самых популярных и быстрых брокеров сообщений. Его используют Netflix, Uber, LinkedIn и тысячи других компаний.

Особенности Kafka:

**1. Журнал коммитов (commit log).** Сообщения не удаляются после чтения — они хранятся в append-only журнале. Можно перечитать сообщение повторно (например, если consumer упал и перезапустился).

```
Topic: "orders"
┌────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ ← offset (номер сообщения)
└────┴────┴────┴────┴────┴────┘
  ▲                         ▲
  │                         │
  старые                    новые
  сообщения                 сообщения
                            (запись только сюда →)
```

**2. Секции (partitions).** Каждая тема делится на секции для параллелизма. Сообщения внутри секции упорядочены. Секции могут находиться на разных серверах.

```
Topic: "orders" (3 partitions)

  Partition 0: [msg0, msg3, msg6, msg9 ...]
  Partition 1: [msg1, msg4, msg7, msg10...]
  Partition 2: [msg2, msg5, msg8, msg11...]
```

**3. Группы потребителей (consumer groups).** Потребители объединяются в группы. Из одной секции читает **только один** потребитель в группе. Это обеспечивает параллельную обработку без дублирования.

```
Topic "orders" (3 partitions)

  Consumer Group "kitchen"
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ Consumer 1 │  │ Consumer 2 │  │ Consumer 3 │
  │ ← Part. 0  │  │ ← Part. 1  │  │ ← Part. 2  │
  └────────────┘  └────────────┘  └────────────┘
```

> **Что изменилось по сравнению с оригинальным уроком:**
> - Zookeeper **больше не нужен**. Начиная с Kafka 3.3 (2022) Kafka работает в режиме **KRaft** (Kafka Raft) — управляет метаданными сам, без внешнего Zookeeper. С Kafka 4.0 (2024) Zookeeper удалён полностью.
> - Установка через **Docker**, а не распаковка архива на Windows.
> - Используем официальный образ `apache/kafka:latest`.

---

## Установка Kafka через Docker

### Предварительные требования

- Docker и Docker Compose установлены. Проверьте:

```bash
docker --version
docker compose version
```

### docker-compose.yml

Создайте файл `docker-compose.yml`:

```yaml
services:
  broker:
    image: apache/kafka:latest
    container_name: broker
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@broker:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

Разберём ключевые параметры:
- `KAFKA_PROCESS_ROLES: broker,controller` — KRaft режим. Один узел выполняет роли и брокера (обработка сообщений), и контроллера (управление кластером). Zookeeper не нужен.
- `KAFKA_LISTENERS` — два слушателя: `PLAINTEXT` на порту 9092 для клиентов, `CONTROLLER` на 9093 для внутренней координации.
- `KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092` — адрес, по которому клиенты (Spring Boot) подключаются к Kafka.
- `KAFKA_CONTROLLER_QUORUM_VOTERS: 1@broker:9093` — список контроллеров для голосования (KRaft). У нас один узел.

### Запуск

```bash
# Запустить Kafka в фоне
docker compose up -d

# Проверить, что контейнер работает
docker compose ps

# Посмотреть логи (должно быть "Kafka Server started")
docker compose logs broker | grep -i started
```

### Создание темы

```bash
docker exec broker \
  /opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic job4j-events \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1
```

### Отправка сообщений (producer)

Откройте **первый** терминал:

```bash
docker exec -it broker \
  /opt/kafka/bin/kafka-console-producer.sh \
  --topic job4j-events \
  --bootstrap-server localhost:9092
```

Вводите текст и нажимайте Enter — каждая строка станет сообщением.

### Чтение сообщений (consumer)

Откройте **второй** терминал:

```bash
docker exec -it broker \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --topic job4j-events \
  --from-beginning \
  --bootstrap-server localhost:9092
```

Вы увидите все отправленные сообщения. Отправьте ещё — они появятся в реальном времени.

### Остановка

```bash
docker compose down        # остановить и удалить контейнеры
docker compose down -v     # + удалить данные (volume)
```

---

## Бонус: Kafka UI

Для визуального мониторинга можно добавить [Kafka UI](https://github.com/provectus/kafka-ui) в `docker-compose.yml`:

```yaml
services:
  broker:
    # ... (как выше)

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8088:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: broker:9092
    depends_on:
      - broker
```

После `docker compose up -d` откройте `http://localhost:8088` — веб-интерфейс для просмотра тем, сообщений, consumer groups.

---

## Преимущества и недостатки брокеров сообщений

**Преимущества:**
1. **Гарантированная доставка.** Сообщения хранятся пока не будут доставлены (и даже после — для повторного чтения).
2. **Множественная рассылка.** Одно сообщение — много получателей через подписку на темы.
3. **Асинхронность из коробки.** Событийная модель, сервис не блокируется.
4. **Развязка сервисов.** Отправитель не знает, кто получатель и где он находится.
5. **Скорость.** Kafka обрабатывает миллионы сообщений в секунду.
6. **Масштабируемость.** Добавление брокеров и секций для увеличения пропускной способности.

**Недостатки:**
1. **Операционная сложность.** Ещё одна система для поддержки и мониторинга.
2. **Eventual consistency.** Данные между сервисами не синхронизированы мгновенно.
3. **Отладка.** Асинхронные цепочки сложнее отследить, чем синхронные вызовы.

## Ссылки

- [Apache Kafka — Introduction](https://kafka.apache.org/intro)
- [Apache Kafka Docker Image](https://hub.docker.com/r/apache/kafka)
- [KRaft: Kafka without Zookeeper](https://developer.confluent.io/learn/kraft/)

## Задание

1. Разверните Kafka через Docker с помощью `docker-compose.yml` из этого урока.
2. Создайте тему `job4j-events`, отправьте несколько сообщений через producer и прочитайте их через consumer.
3. *Дополнительно:* добавьте Kafka UI и посмотрите на темы и сообщения через браузер.
4. В комментарии напишите, что система развёрнута успешно и сообщения отправляются и приходят.
