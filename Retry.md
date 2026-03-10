# Шаблон Retry [v2, 2026]

## Введение

Любой ресурс, использующий сеть, может стать недоступным: проблемы с сетью, перезапуск сервиса, временная перегрузка. В микросервисной архитектуре такие ситуации — не исключение, а норма. Шаблон Retry позволяет автоматически повторять неудавшуюся операцию, повышая надёжность системы.

> **Что изменилось по сравнению с оригинальным уроком:**
> - Оригинал давал ручную реализацию Retry через `while` + `Thread.sleep()` — полезно для понимания механики, но в реальных проектах так не делают.
> - В 2026 году есть три готовых решения: **Spring Retry** (библиотека), **Spring Framework 7 native @Retryable** (встроено в ядро), и **Resilience4j** (продвинутые паттерны). Мы рассмотрим все три.

---

## Зачем нужен Retry

```
Сервис A                          Сервис B
    │                                 │
    │── GET /api/data ───────────────▶│
    │                                 │  ✗ Сервис перезапускается
    │◀── 503 Service Unavailable ─────│
    │                                 │
    │   ⏳ ждём 1 секунду             │
    │                                 │
    │── GET /api/data ───────────────▶│
    │                                 │  ✗ Ещё не готов
    │◀── 503 Service Unavailable ─────│
    │                                 │
    │   ⏳ ждём 2 секунды             │
    │                                 │
    │── GET /api/data ───────────────▶│
    │                                 │  ✓ Готов!
    │◀── 200 OK + данные ────────────│
```

Без Retry клиент сразу получил бы ошибку 503. С Retry система восстановилась самостоятельно.

### Основные параметры Retry

1. **Количество попыток** — сколько раз повторять (обычно 3–5).
2. **Задержка между попытками** — фиксированная или экспоненциальная.
3. **Какие ошибки повторять** — не все ошибки имеет смысл ретраить (например, 404 ретраить бессмысленно).
4. **Fallback (восстановление)** — что делать, если все попытки исчерпаны.

---

## Уровень 0: Ручная реализация (для понимания)

Это примерно то, что было в оригинальном уроке. Полезно, чтобы понять механику, но **никогда не используйте это в продакшене**.

```java
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@AllArgsConstructor
public class Retry {
    private final int retries;
    private final long delay;

    public interface Act<T> {
        T exec() throws Exception;
    }

    public <R> R exec(Act<R> act, R defVal) {
        int i = 0;
        do {
            i++;
            try {
                return act.exec();
            } catch (Exception e) {
                log.error("Attempt {} failed: {}", i, e.getMessage());
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        } while (i < retries);
        return defVal;
    }
}
```

### Проблемы ручной реализации

- **Фиксированная задержка** — при массовых сбоях все клиенты ретраят одновременно, создавая «толпу» (thundering herd).
- **Нет экспоненциального backoff** — задержка не увеличивается с каждой попыткой.
- **Нет jitter** — нет случайного разброса задержки.
- **Thread.sleep() блокирует поток** — неэффективно.
- **Нет различения типов ошибок** — ретраит всё подряд, включая 404.
- **Нет fallback** — при исчерпании попыток просто возвращается дефолтное значение.
- **Не интегрирован со Spring** — не работает как AOP-аспект.

Все эти проблемы решены в Spring Retry.

---

## Уровень 1: Spring Retry (библиотека)

Это решение работает со **Spring Boot 3.x** — текущим стабильным релизом в 2026 году.

### Зависимости

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

> `spring-boot-starter-aop` нужен, потому что `@Retryable` работает через AOP-прокси — Spring перехватывает вызов метода и оборачивает его в логику повтора.

### Включение

```java
package ru.job4j.notification.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.retry.annotation.EnableRetry;

@Configuration
@EnableRetry
public class RetryConfig {
}
```

### Декларативный подход: @Retryable + @Recover

```java
package ru.job4j.notification.service;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientException;
import ru.job4j.notification.model.User;

@Slf4j
@Service
@AllArgsConstructor
public class AuthService {

    private final RestClient restClient;

    @Retryable(
            retryFor = RestClientException.class,   // какие ошибки ретраить
            noRetryFor = IllegalArgumentException.class, // какие НЕ ретраить
            maxAttempts = 3,                         // всего 3 попытки (включая первую)
            backoff = @Backoff(
                    delay = 1000,       // начальная задержка 1 секунда
                    multiplier = 2.0,   // каждая следующая задержка × 2
                    maxDelay = 10000    // максимум 10 секунд
            )
    )
    public User findUser(int id) {
        log.info("Попытка получить пользователя с id={}", id);
        return restClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .body(User.class);
    }

    /**
     * Fallback: вызывается, если все попытки исчерпаны.
     * Сигнатура: тот же тип возврата + первый аргумент — пойманное исключение.
     */
    @Recover
    public User recoverFindUser(RestClientException ex, int id) {
        log.warn("Все попытки получить пользователя id={} исчерпаны: {}",
                id, ex.getMessage());
        return null;  // или вернуть закешированного пользователя
    }
}
```

Что здесь происходит:

```
findUser(42) — 1-я попытка
    │
    ✗ RestClientException → ждём 1 сек
    │
findUser(42) — 2-я попытка
    │
    ✗ RestClientException → ждём 2 сек (× 2)
    │
findUser(42) — 3-я попытка
    │
    ✗ RestClientException → все попытки исчерпаны
    │
recoverFindUser(ex, 42) — fallback
```

> **Экспоненциальный backoff** — задержки растут: 1с → 2с → 4с → ... Это даёт перегруженному сервису время на восстановление, вместо того чтобы долбить его каждую секунду.

### Императивный подход: RetryTemplate

Если вам нужен retry для конкретного блока кода, а не для всего метода:

```java
package ru.job4j.notification.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.retry.support.RetryTemplate;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientException;
import ru.job4j.notification.model.User;

@Slf4j
@Service
public class AuthServiceWithTemplate {

    private final RestClient restClient;
    private final RetryTemplate retryTemplate;

    public AuthServiceWithTemplate(RestClient restClient) {
        this.restClient = restClient;
        this.retryTemplate = RetryTemplate.builder()
                .maxAttempts(3)
                .exponentialBackoff(1000, 2.0, 10000)
                .retryOn(RestClientException.class)
                .build();
    }

    public User findUser(int id) {
        return retryTemplate.execute(context -> {
            log.info("Попытка #{} получить пользователя id={}",
                    context.getRetryCount() + 1, id);
            return restClient.get()
                    .uri("/users/{id}", id)
                    .retrieve()
                    .body(User.class);
        });
    }
}
```

`RetryTemplate.builder()` — fluent API, появившийся в Spring Retry 2.0. Сравните с оригинальным ручным `while` + `Thread.sleep()` — тот же результат, но надёжнее и лаконичнее.

---

## Уровень 2: Native @Retryable (Spring Framework 7 / Spring Boot 4)

В сентябре 2025 года Spring Framework 7 представил **встроенную поддержку Retry** прямо в ядре фреймворка — без внешней зависимости `spring-retry`.

> Это новый подход, который стоит знать, хотя в курсе мы работаем со Spring Boot 3.4. Если ваш проект уже на Spring Boot 4 — используйте нативный вариант.

```java
import org.springframework.retry.annotation.Retryable;  // это уже из spring-core!

@Service
public class AuthService {

    @Retryable(maxAttempts = 3, delay = 1000)
    public User findUser(int id) {
        return restClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .body(User.class);
    }
}
```

Отличия от Spring Retry (библиотеки):

| Критерий | Spring Retry (библиотека) | Spring Framework 7 (native) |
|----------|--------------------------|----------------------------|
| **Зависимость** | `spring-retry` + `spring-boot-starter-aop` | Встроено в `spring-context` |
| **Активация** | `@EnableRetry` | `@EnableResilientMethods` или `RetryAnnotationBeanPostProcessor` |
| **maxAttempts** | Общее число попыток (включая первую) | Число **повторных** попыток (без первой) |
| **Реактивность** | Нет | Поддерживает `Mono`/`Flux` из коробки |

---

## Уровень 3: Resilience4j (упоминание)

Для продвинутых сценариев, где одного Retry недостаточно, существует библиотека **Resilience4j**. Она предоставляет комбинацию паттернов:

- **Retry** — повторные попытки
- **Circuit Breaker** — «автоматический выключатель», который перестаёт отправлять запросы, если сервис упал надолго
- **Rate Limiter** — ограничение частоты запросов
- **Bulkhead** — изоляция ресурсов

Конфигурируется через `application.yml`:

```yaml
resilience4j:
  retry:
    instances:
      authService:
        maxAttempts: 4
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.client.RestClientException
```

Resilience4j — тема отдельного урока. Здесь упоминаем, чтобы вы знали: Retry — это лишь один из паттернов отказоустойчивости.

---

## Сравнение подходов

| Критерий | Ручной Retry | Spring Retry | SF 7 Native | Resilience4j |
|----------|-------------|-------------|-------------|-------------|
| **Зависимости** | Нет | `spring-retry` + AOP | Встроено | `resilience4j-spring-boot3` |
| **Backoff** | Только фиксированный | Фиксированный, экспоненциальный | Фиксированный, экспоненциальный | + jitter из коробки |
| **Fallback** | Дефолтное значение | `@Recover` | — | Fallback-метод |
| **Фильтрация ошибок** | Нет | `retryFor` / `noRetryFor` | `includes` / `excludes` | `retryExceptions` / `ignoreExceptions` |
| **Circuit Breaker** | Нет | Нет | Нет | Да |
| **Реактивность** | Нет | Нет | Да (`Mono`/`Flux`) | Да |
| **Для продакшена** | ❌ | ✅ | ✅ | ✅ |

---

## Когда что использовать

**Учебный проект, понять механику** → ручная реализация (как в оригинальном уроке).

**Spring Boot 3.x, типичный микросервис** → Spring Retry с `@Retryable`. Минимум кода, декларативный подход, экспоненциальный backoff.

**Spring Boot 4.x** → нативный `@Retryable` из Spring Framework 7, без внешних зависимостей.

**Высоконагруженный продакшен, нужны Circuit Breaker + Rate Limiter** → Resilience4j.

---

## Важные практические моменты

### Не ретрайте всё подряд

```java
// ❌ ПЛОХО — ретраит даже 404, 400, 422
@Retryable(retryFor = Exception.class)

// ✅ ХОРОШО — ретраит только сетевые/серверные ошибки
@Retryable(
    retryFor = {RestClientException.class, ResourceAccessException.class},
    noRetryFor = {HttpClientErrorException.NotFound.class}
)
```

### Идемпотентность

Retry безопасен только для **идемпотентных** операций — тех, которые можно вызвать повторно без побочных эффектов. `GET` — идемпотентный. `POST` создания заказа — нет (можно создать дубль). Прежде чем ретраить `POST`, убедитесь, что сервер обрабатывает дубликаты корректно (например, через ключ идемпотентности).

### Экспоненциальный backoff + jitter

Фиксированная задержка в 1 секунду при массовом сбое означает, что все 1000 клиентов ретраят одновременно через 1 секунду — это «толпа» (thundering herd). Экспоненциальный backoff с jitter (случайным разбросом) распределяет нагрузку:

```
Клиент A: 1.2с → 2.4с → 5.1с
Клиент B: 0.8с → 1.7с → 3.9с
Клиент C: 1.0с → 2.1с → 4.3с
```

---

## Ссылки

- [Spring Retry — GitHub](https://github.com/spring-projects/spring-retry)
- [Guide to Spring Retry — Baeldung](https://www.baeldung.com/spring-retry)
- [Core Spring Resilience Features — Spring Blog](https://spring.io/blog/2025/09/09/core-spring-resilience-features/) — анонс нативного @Retryable в Spring Framework 7
- [Resilience4j](https://resilience4j.readme.io/)

---

## Задание

1. Реализуйте шаблон Retry в сервисе notification с использованием **Spring Retry** (`@Retryable` + `@Recover`). Вызовы сервиса auth должны быть выполнены через `@Retryable` с экспоненциальным backoff.
2. *Дополнительно:* реализуйте тот же Retry через `RetryTemplate` (императивный подход) и сравните оба варианта.
3. Загрузите изменения в репозиторий. Оставьте ссылку.
