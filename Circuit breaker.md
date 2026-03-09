# Resilience4j [v2, 2026]

## Введение

В предыдущих уроках мы разобрали шаблоны Retry и Circuit Breaker — сначала ручные реализации для понимания механики, затем Spring Retry для Retry. Теперь объединим оба паттерна через **Resilience4j** — стандартную библиотеку отказоустойчивости для Spring Boot 3.x.

> **Что изменилось по сравнению с оригинальным уроком:**
> - `resilience4j-spring-boot2` → `resilience4j-spring-boot3` (для Spring Boot 3.x)
> - Версия `1.7.0` → `2.2.0`
> - Добавлена обязательная зависимость `spring-boot-starter-aop` (без неё аннотации не работают)
> - Добавлен `spring-boot-starter-actuator` для мониторинга
> - Объяснён порядок выполнения Retry и Circuit Breaker (это критично)
> - Разобрана ловушка Resilience4j + реактивные типы (`Mono`/`Flux`)
> - Конфигурация показана в `application.yml` (нагляднее, чем `.properties`)

---

## Зависимости

```xml
<!-- Resilience4j: Spring Boot 3 Starter -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>

<!-- AOP: ОБЯЗАТЕЛЬНО — без него @Retry и @CircuitBreaker не работают -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<!-- Actuator: мониторинг состояния Circuit Breaker -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Reactor: для работы @CircuitBreaker с Mono/Flux -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
    <version>2.2.0</version>
</dependency>
```

> **Частая ошибка:** подключить `resilience4j-spring-boot3` без `spring-boot-starter-aop`. Приложение запустится, аннотации `@Retry` и `@CircuitBreaker` будут компилироваться, но **ничего не произойдёт** — вызовы будут идти напрямую, без ретраев и без circuit breaker. Это потому что аннотации работают через AOP-прокси, а без `starter-aop` прокси не создаётся.
>
> **Вторая ошибка:** не подключить `resilience4j-reactor` при работе с `Mono`/`Flux`. Без него Resilience4j не умеет корректно оборачивать реактивные типы.

---

## Конфигурация

### application.yml

```yaml
resilience4j:
  retry:
    instances:
      authService:
        maxAttempts: 3                    # 3 попытки (включая первую)
        waitDuration: 500ms               # 500мс между попытками
        enableExponentialBackoff: true     # экспоненциальный рост задержки
        exponentialBackoffMultiplier: 2    # каждая следующая × 2
        retryExceptions:                  # ретраить только эти ошибки
          - org.springframework.web.reactive.function.client.WebClientRequestException
          - java.util.concurrent.TimeoutException
    # Порядок: Retry оборачивает CircuitBreaker
    retryAspectOrder: 2

  circuitbreaker:
    instances:
      authService:
        slidingWindowSize: 10                          # последние 10 вызовов
        minimumNumberOfCalls: 5                        # начинать оценку после 5
        failureRateThreshold: 50                       # 50% ошибок → OPEN
        waitDurationInOpenState: 30s                   # в OPEN ждём 30 сек
        permittedNumberOfCallsInHalfOpenState: 3       # в HALF-OPEN пропускаем 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        registerHealthIndicator: true
    circuitBreakerAspectOrder: 1

# Actuator: мониторинг
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

### Разбор ключевых параметров

**Retry:**
- `maxAttempts: 3` — максимум 3 попытки. Если все 3 провалились — ошибка уходит дальше (к Circuit Breaker).
- `waitDuration: 500ms` — начальная пауза между попытками.
- `enableExponentialBackoff: true` + `multiplier: 2` — задержки растут: 500мс → 1с → 2с. Это даёт перегруженному сервису время на восстановление.
- `retryExceptions` — ретраим только сетевые ошибки. Нет смысла ретраить 404 или 400.

**Circuit Breaker:**
- `slidingWindowSize: 10` — анализируем **последние 10 вызовов**, а не все с начала работы.
- `failureRateThreshold: 50` — если 50%+ вызовов в окне неуспешны → переход в OPEN.
- `waitDurationInOpenState: 30s` — в OPEN не отправляем запросы 30 секунд, затем переход в HALF-OPEN.
- `permittedNumberOfCallsInHalfOpenState: 3` — в HALF-OPEN пропускаем 3 пробных запроса.

### Порядок выполнения (критично!)

```yaml
resilience4j:
  retry:
    retryAspectOrder: 2          # выше = раньше (внешний слой)
  circuitbreaker:
    circuitBreakerAspectOrder: 1  # ниже = позже (внутренний слой)
```

Это означает:

```
Запрос → [Retry (order=2)] → [CircuitBreaker (order=1)] → WebClient → Auth
             внешний                 внутренний
```

Почему именно так:
- Retry оборачивает Circuit Breaker: неудачный вызов **сначала** проверяется CB (открыт ли?), и только потом ретраится.
- Если CB в состоянии OPEN — Retry получит `CallNotPermittedException` мгновенно. Не будет ждать таймаут сети.
- Три ретрая неудачного вызова считаются **одной** ошибкой для CB (а не тремя).

Если сделать наоборот (CB снаружи, Retry внутри) — каждая попытка Retry будет отдельным вызовом для CB. Один провальный запрос запишет сразу 3 ошибки в скользящее окно, и CB откроется слишком рано.

---

## Код сервиса

```java
package ru.job4j.notification.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import ru.job4j.notification.model.PersonDTO;

@Service
@Slf4j
public class AuthCallWebClient {

    private final WebClient webClient;

    public AuthCallWebClient(@Value("${server.auth}") String urlAuth) {
        this.webClient = WebClient.builder()
                .baseUrl(urlAuth)
                .build();
    }

    /**
     * GET-запрос к сервису auth.
     *
     * Порядок выполнения:
     * 1. Retry (order=2) — если вызов провалился, повторяем до 3 раз
     * 2. CircuitBreaker (order=1) — если CB открыт, сразу fallback
     * 3. WebClient — собственно HTTP-вызов
     */
    @Retry(name = "authService")
    @CircuitBreaker(name = "authService", fallbackMethod = "fallbackGet")
    public Mono<PersonDTO> doGet(String url) {
        return webClient.get()
                .uri(url)
                .retrieve()
                .bodyToMono(PersonDTO.class);
    }

    /**
     * POST-запрос к сервису auth.
     */
    @Retry(name = "authService")
    @CircuitBreaker(name = "authService", fallbackMethod = "fallbackPost")
    public Mono<Object> doPost(String url, PersonDTO personDTO) {
        return webClient.post()
                .uri(url)
                .bodyValue(personDTO)
                .retrieve()
                .bodyToMono(Object.class);
    }

    /**
     * Fallback для GET.
     * Сигнатура: те же аргументы + Throwable в конце.
     */
    private Mono<PersonDTO> fallbackGet(String url, Throwable ex) {
        log.warn("Auth GET {} — fallback: {}", url, ex.getMessage());
        return Mono.empty();
    }

    /**
     * Fallback для POST.
     */
    private Mono<Object> fallbackPost(String url, PersonDTO personDTO,
                                       Throwable ex) {
        log.warn("Auth POST {} — fallback: {}", url, ex.getMessage());
        return Mono.empty();
    }
}
```

> **Обратите внимание:** fallback-методы объявлены как `private`. Это правильно — они не должны вызываться напрямую, только через AOP-прокси Resilience4j. Fallback-метод должен иметь **ту же сигнатуру** (те же аргументы), плюс `Throwable` последним параметром.

---

## Ловушка: Resilience4j + Mono/Flux

Есть важный нюанс при использовании `@CircuitBreaker` и `@Retry` с реактивными типами.

Аннотации Resilience4j работают через AOP — перехватывают **вызов метода**. Но `Mono` и `Flux` — ленивые: сам HTTP-запрос выполняется не в момент вызова метода, а когда кто-то подпишется на `Mono`. Это значит, что без `resilience4j-reactor` ошибка произойдёт **после** того, как AOP-перехватчик уже отработал — и ни Retry, ни Circuit Breaker её не увидят.

Именно поэтому нужна зависимость:

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
    <version>2.2.0</version>
</dependency>
```

Она добавляет поддержку реактивных типов: Resilience4j оборачивает сам `Mono`/`Flux` (через оператор `transformDeferred`), а не только вызов метода. Без этой зависимости аннотации с реактивными возвращаемыми типами **молча не работают**.

> **Правило:** если метод возвращает `Mono` или `Flux` — обязательно подключите `resilience4j-reactor`. Если метод возвращает обычный объект (синхронный вызов через `RestClient`) — `resilience4j-reactor` не нужен.

---

## Как это работает в действии

```
doGet("/api/users/42")
         │
    ┌────▼────┐
    │  Retry   │  попытка 1
    │ (order=2)│──────┐
    └─────────┘      │
                ┌────▼─────────┐
                │ CircuitBreaker│  state = CLOSED → пропускаем
                │  (order=1)    │──────┐
                └──────────────┘      │
                                 ┌────▼────┐
                                 │ WebClient│──▶ Auth Service
                                 └─────────┘
                                      │
                                      ✗ 503 Service Unavailable
                                      │
    ┌─────────┐                       │
    │  Retry   │  попытка 2 (через 500мс)
    │          │──────┐
    └─────────┘      │
                ┌────▼─────────┐
                │ CircuitBreaker│  state = CLOSED → пропускаем
                └──────┬───────┘
                       │
                       ✗ 503 снова
                       │
    ┌─────────┐        │
    │  Retry   │  попытка 3 (через 1с)
    │          │──────┐
    └─────────┘      │
                ┌────▼─────────┐
                │ CircuitBreaker│  state = CLOSED → пропускаем
                └──────┬───────┘
                       │
                       ✗ 503 снова
                       │
    Retry исчерпан → ошибка записывается в CB как 1 неудачный вызов
                       │
    ... после нескольких таких серий (50% ошибок в окне 10) ...
                       │
    ┌─────────┐        │
    │  Retry   │  следующий вызов
    │          │──────┐
    └─────────┘      │
                ┌────▼─────────┐
                │ CircuitBreaker│  state = OPEN → мгновенный отказ!
                └──────┬───────┘
                       │
                       ▼
                fallbackGet() → Mono.empty()
                (НЕ ждём 30 секунд таймаута сети)
```

---

## Мониторинг через Actuator

После настройки состояние Circuit Breaker доступно по адресу:

```
GET http://localhost:8080/actuator/health
```

Ответ при нормальной работе:

```json
{
  "components": {
    "circuitBreakers": {
      "details": {
        "authService": {
          "state": "CLOSED",
          "failureRate": "-1.0%",
          "bufferedCalls": 3,
          "failedCalls": 0
        }
      }
    }
  }
}
```

> `failureRate: -1.0%` означает «ещё не набрано `minimumNumberOfCalls` (5) для оценки». После 5 вызовов появится реальный процент.

Ответ при сбое auth:

```json
{
  "authService": {
    "state": "OPEN",
    "failureRate": "60.0%",
    "bufferedCalls": 10,
    "failedCalls": 6
  }
}
```

Это позволяет настроить алерт: если `state: OPEN` — auth лежит, пора разбираться.

---

## Что изменилось для вызывающего кода?

**Ничего.** В этом и красота подхода. Код, который вызывает `authCallWebClient.doGet("/api/users/42")`, не знает ни про Retry, ни про Circuit Breaker. Он просто получает `Mono<PersonDTO>` — либо с данными, либо пустой (fallback). Вся отказоустойчивость — внутри аннотаций и конфигурации.

```java
// Вызывающий код — НЕ меняется
authCallWebClient.doGet("/api/users/" + userId)
        .subscribe(user -> sendNotification(user));
```

---

## Сравнение: было и стало

| Критерий | Ручной Retry + CB | Resilience4j |
|----------|------------------|-------------|
| **Код сервиса** | `new Retry(3, 1000)` + `new CircuitBreaker(2)` вручную | `@Retry` + `@CircuitBreaker` |
| **Конфигурация** | Хардкод в коде | `application.yml` (меняется без перекомпиляции) |
| **Half-Open** | Нет | Есть, автоматический |
| **Экспоненциальный backoff** | Нет | `enableExponentialBackoff: true` |
| **Fallback** | Дефолтное значение | Отдельный метод |
| **Мониторинг** | Нет | Actuator + `/health` |
| **Реактивность** | Блокирующий `Thread.sleep()` | Нативная поддержка `Mono`/`Flux` |
| **Влияние на вызывающий код** | Нет | Нет |

---

## Ссылки

- [Resilience4j — Getting Started (Spring Boot 3)](https://resilience4j.readme.io/docs/getting-started-3) — официальная документация
- [Guide to Resilience4j with Spring Boot — Baeldung](https://www.baeldung.com/spring-boot-resilience4j)
- [Resilience4j — Circuit Breaker](https://resilience4j.readme.io/docs/circuitbreaker) — детальная документация по CB
- [Resilience4j — Retry](https://resilience4j.readme.io/docs/retry) — детальная документация по Retry

---

## Задание

1. Подключите к сервису notification библиотеку Resilience4j (`resilience4j-spring-boot3` + `resilience4j-reactor` + `starter-aop` + `starter-actuator`).
2. Настройте Retry и Circuit Breaker для вызовов auth в `application.yml` с параметрами из этого урока.
3. Замените ручные `new Retry()` и `new CircuitBreaker()` на аннотации `@Retry` и `@CircuitBreaker`.
4. Убедитесь, что порядок выполнения правильный: Retry снаружи (`order=2`), Circuit Breaker внутри (`order=1`).
5. *Дополнительно:* проверьте, что состояние CB отображается в `/actuator/health`, и смоделируйте переход в OPEN (например, остановите сервис auth).
6. Загрузите изменения в репозиторий. Оставьте ссылку на коммит.
