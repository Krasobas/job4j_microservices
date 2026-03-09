# Circuit Breaker [v2, 2026]

## Введение

В предыдущем уроке мы разобрали шаблон Retry — повторные попытки вызова. Но Retry одного недостаточно. Представьте: сервис не просто «упал на секунду» — он завис и отвечает по 30 секунд. Retry в этом случае только усугубит ситуацию: мы будем 3 раза ждать по 30 секунд, блокируя потоки. А если таких запросов сотни — система встанет целиком.

Для таких случаев существует шаблон **Circuit Breaker** — «автоматический выключатель».

> **Что изменилось по сравнению с оригинальным уроком:**
> - Оригинал давал ручную реализацию Circuit Breaker, которая работала только на закрытие (без состояния Half-Open) и не идентифицировала сервисы.
> - В этом уроке мы сохраняем ручную реализацию для понимания, а затем показываем **Resilience4j** — стандартную библиотеку для отказоустойчивости в Spring Boot 3.x.

---

## Аналогия

Circuit Breaker работает как автомат защиты в электрическом щитке. Когда ток слишком большой (сервис сбоит) — автомат выбивает (circuit открывается). Электричество перестаёт течь (запросы не уходят). Через какое-то время вы пробуете включить автомат обратно (half-open). Если всё в порядке — работаем дальше. Если опять выбило — ждём ещё.

---

## Три состояния

```
                    ошибок < порога
              ┌────────────────────────┐
              │                        │
              ▼                        │
         ┌─────────┐            ┌─────────────┐
         │ CLOSED  │───────────▶│    OPEN      │
         │         │  ошибок    │             │
         │ Всё ОК, │  >= порога │ Запросы НЕ  │
         │ запросы  │            │ отправляются│
         │ идут     │            │ (fail fast) │
         └─────────┘            └──────┬──────┘
              ▲                        │
              │                        │ таймаут
              │   запросы              │ истёк
              │   успешны              ▼
              │                 ┌─────────────┐
              └────────────────│  HALF-OPEN   │
                               │             │
                               │ Пропускаем  │
                               │ несколько   │──── ошибки? ──▶ обратно в OPEN
                               │ пробных     │
                               │ запросов    │
                               └─────────────┘
```

**CLOSED** — нормальная работа. Запросы проходят. Если количество ошибок в скользящем окне превышает порог — переход в OPEN.

**OPEN** — «выключатель сработал». Запросы **не отправляются** вообще — сразу возвращается ошибка или fallback. Система не тратит ресурсы на заведомо неработающий сервис. После заданного таймаута — переход в HALF-OPEN.

**HALF-OPEN** — «пробуем включить обратно». Пропускаем несколько пробных запросов. Если они успешны — возвращаемся в CLOSED. Если снова ошибки — обратно в OPEN.

---

## Retry vs Circuit Breaker

Эти два паттерна не заменяют, а **дополняют** друг друга:

```
Запрос → [Retry] → [Circuit Breaker] → Сервис
```

| Ситуация | Retry | Circuit Breaker |
|----------|-------|----------------|
| Сервис моргнул на 1 секунду | ✅ Повторил и получил ответ | Не нужен |
| Сервис лежит 5 минут | ❌ 3 попытки × 30сек = 1.5мин ожидания | ✅ Сразу отказ, не ждём |
| Сервис перегружен | ❌ Retry добавляет нагрузку | ✅ Даёт время на восстановление |

Правило: **Retry без Circuit Breaker опасен** — если сервис перегружен, ретраи только усиливают нагрузку на него. Всегда комбинируйте.

---

## Уровень 0: Ручная реализация (для понимания)

Это упрощённая версия из оригинального урока. Полезна для понимания механики, но **не используйте в продакшене**.

```java
package ru.job4j.notification.service;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class CircuitBreaker {
    private final int failureThreshold;
    private int failureCount = 0;
    private State state = State.CLOSED;

    public CircuitBreaker(int failureThreshold) {
        this.failureThreshold = failureThreshold;
    }

    private enum State {
        OPEN,
        CLOSED
    }

    public interface Act<T> {
        T exec() throws Exception;
    }

    public <R> R exec(Act<R> act, R defVal) {
        if (state == State.CLOSED) {
            try {
                return act.exec();
            } catch (Exception e) {
                failureCount++;
                log.error("Attempt failed, failure count: {}", failureCount);
                if (failureCount >= failureThreshold) {
                    state = State.OPEN;
                    log.warn("Circuit Breaker OPENED.");
                }
                return defVal;
            }
        }
        log.warn("Circuit Breaker is OPEN. Skipping request.");
        throw new CircuitBreakerOpenException(
                "Circuit Breaker is OPEN. Request skipped."
        );
    }

    public static class CircuitBreakerOpenException extends RuntimeException {
        public CircuitBreakerOpenException(String message) {
            super(message);
        }
    }
}
```

### Что не так с ручной реализацией

- **Нет состояния HALF-OPEN** — раз открылся, так и остаётся открытым навсегда. Нет механизма «попробовать снова».
- **Нет скользящего окна** — считает ошибки с начала жизни объекта, а не за последние N вызовов.
- **Не потокобезопасно** — `failureCount++` без синхронизации.
- **Не различает сервисы** — один экземпляр на все вызовы.
- **Нет fallback** — просто бросает исключение.
- **Нет метрик** — невозможно мониторить состояние.

Все эти проблемы решены в Resilience4j.

---

## Уровень 1: Resilience4j (продакшен-подход)

**Resilience4j** — стандартная библиотека для отказоустойчивости в Spring Boot 3.x. Она пришла на замену Netflix Hystrix (который давно в режиме maintenance). Resilience4j модульная — вы подключаете только те паттерны, которые нужны.

### Зависимости

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

> `spring-boot-starter-aop` нужен для аннотации `@CircuitBreaker` (работает через AOP-прокси). `spring-boot-starter-actuator` — для мониторинга состояния Circuit Breaker через `/actuator/health`.

### Конфигурация в application.yml

```yaml
resilience4j:
  circuitbreaker:
    instances:
      authService:                          # имя — используется в аннотации
        slidingWindowSize: 10               # анализируем последние 10 вызовов
        minimumNumberOfCalls: 5             # начинаем оценивать после 5 вызовов
        failureRateThreshold: 50            # порог ошибок: 50% → открыть
        waitDurationInOpenState: 30s        # в OPEN ждём 30 секунд
        permittedNumberOfCallsInHalfOpenState: 3  # в HALF-OPEN пропускаем 3 пробных
        automaticTransitionFromOpenToHalfOpenEnabled: true  # авто-переход в HALF-OPEN
        registerHealthIndicator: true       # видно в /actuator/health

management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

Разберём ключевые параметры:

- `slidingWindowSize: 10` — Circuit Breaker анализирует **последние 10 вызовов** (скользящее окно). Это не «10 ошибок с начала работы», а «из последних 10 вызовов сколько было ошибок».
- `failureRateThreshold: 50` — если 50% и более вызовов в окне неуспешны — переход в OPEN.
- `waitDurationInOpenState: 30s` — в состоянии OPEN ждём 30 секунд перед переходом в HALF-OPEN.
- `permittedNumberOfCallsInHalfOpenState: 3` — в HALF-OPEN пропускаем 3 пробных запроса. Если они успешны — CLOSED. Если нет — обратно в OPEN.

### Использование в коде

```java
package ru.job4j.notification.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import ru.job4j.notification.model.User;

@Slf4j
@Service
@AllArgsConstructor
public class AuthService {

    private final RestClient restClient;

    @CircuitBreaker(name = "authService", fallbackMethod = "findUserFallback")
    public User findUser(int id) {
        return restClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .body(User.class);
    }

    /**
     * Fallback: вызывается, когда Circuit Breaker в состоянии OPEN
     * или когда вызов завершился ошибкой.
     *
     * Сигнатура: тот же тип возврата + Exception как первый аргумент.
     */
    private User findUserFallback(int id, Throwable ex) {
        log.warn("Circuit Breaker сработал для user id={}: {}", id, ex.getMessage());
        return null;  /* или вернуть закешированного пользователя */
    }
}
```

Что здесь происходит:

```
findUser(42) — 1-й вызов (CLOSED)
    ✓ OK
findUser(42) — 2-й вызов
    ✗ Ошибка (1 из 10)
findUser(42) — 3-й вызов
    ✗ Ошибка (2 из 10)
    ... (ещё ошибки)
findUser(42) — 10-й вызов
    ✗ Ошибка (6 из 10 = 60% > 50%)
    │
    ▼
    Circuit Breaker → OPEN
    │
findUser(42) — 11-й вызов
    │ НЕ отправляется к auth!
    │ Сразу вызывается findUserFallback(42, CallNotPermittedException)
    │
    ⏳ ... 30 секунд ...
    │
    ▼
    Circuit Breaker → HALF-OPEN
    │
findUser(42) — пробный вызов 1 из 3
    ✓ OK
findUser(42) — пробный вызов 2 из 3
    ✓ OK
findUser(42) — пробный вызов 3 из 3
    ✓ OK → Circuit Breaker → CLOSED (восстановлен!)
```

### Комбинирование с Retry

Retry и Circuit Breaker можно использовать вместе. Порядок важен: **сначала Retry, затем Circuit Breaker** — то есть неудачный вызов сначала ретраится, и только если все попытки провалились, это считается одной ошибкой для Circuit Breaker.

```java
@CircuitBreaker(name = "authService", fallbackMethod = "findUserFallback")
@Retry(name = "authService")
public User findUser(int id) {
    return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .body(User.class);
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      authService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
  retry:
    instances:
      authService:
        maxAttempts: 3
        waitDuration: 1s

  # Порядок: Retry оборачивает CircuitBreaker
  # (больше = выше приоритет = выполняется раньше)
  retry:
    retryAspectOrder: 2
  circuitbreaker:
    circuitBreakerAspectOrder: 1
```

```
Запрос → Retry (3 попытки) → Circuit Breaker → RestClient → Auth Service
                                    │
                           если OPEN → fallback (мгновенно)
```

---

## Мониторинг

Одно из ключевых преимуществ Resilience4j — встроенный мониторинг через Spring Boot Actuator.

После настройки (см. конфигурацию выше) состояние Circuit Breaker доступно по адресу:

```
GET http://localhost:8080/actuator/health
```

```json
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "authService": {
          "status": "UP",
          "details": {
            "state": "CLOSED",
            "failureRate": "0.0%",
            "bufferedCalls": 7,
            "failedCalls": 1
          }
        }
      }
    }
  }
}
```

Если Circuit Breaker в состоянии OPEN:

```json
{
  "status": "DOWN",
  "details": {
    "state": "OPEN",
    "failureRate": "60.0%"
  }
}
```

Это позволяет настроить алерты: если `state: OPEN` — значит зависимый сервис лежит, пора разбираться.

---

## Сравнение подходов

| Критерий | Ручная реализация | Resilience4j |
|----------|------------------|-------------|
| **Состояния** | CLOSED, OPEN | CLOSED, OPEN, HALF-OPEN |
| **Скользящее окно** | Нет | COUNT_BASED или TIME_BASED |
| **Потокобезопасность** | Нет | Да (AtomicReference + кольцевой буфер) |
| **Fallback** | Нет (исключение) | `fallbackMethod` в аннотации |
| **Мониторинг** | Нет | Actuator + Micrometer |
| **Конфигурация** | Хардкод | `application.yml` |
| **Комбинирование** | Вручную с Retry | `@CircuitBreaker` + `@Retry` одновременно |
| **Для продакшена** | ❌ | ✅ |

---

## Ссылки

- [Resilience4j — Circuit Breaker](https://resilience4j.readme.io/docs/circuitbreaker) — официальная документация
- [Guide to Resilience4j with Spring Boot — Baeldung](https://www.baeldung.com/spring-boot-resilience4j) — подробный разбор
- [Spring Cloud CircuitBreaker](https://docs.spring.io/spring-cloud-circuitbreaker/docs/current/reference/html/) — интеграция через Spring Cloud

---

## Задание

1. Добавьте Resilience4j в сервис notification. Вызовы к сервису auth должны быть защищены `@CircuitBreaker` с fallback-методом.
2. Настройте параметры Circuit Breaker в `application.yml`: скользящее окно 10, порог ошибок 50%, ожидание в OPEN 30 секунд.
3. Скомбинируйте `@CircuitBreaker` с `@Retry` из предыдущего урока.
4. *Дополнительно:* подключите Actuator и убедитесь, что состояние Circuit Breaker отображается в `/actuator/health`.
5. Загрузите изменения в репозиторий. Оставьте ссылку.
