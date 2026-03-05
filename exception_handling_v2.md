# Обработка исключений в RestClient [v2, 2026]

В этом уроке мы добавим в сервис RestClient обработку ошибок.

> **Примечание.** Оригинальный урок был написан для `RestTemplate`. Начиная с Spring Framework 7.0 `RestTemplate` официально объявлен устаревшим, а его рекомендованная замена — `RestClient`. В этом уроке мы используем `RestClient` и его встроенный механизм обработки ошибок.

## Проблема

Запустим сервис Source и попробуем извлечь объект по несуществующему `id`. Таблица Items пуста, поэтому запрос `getById?id=1` приводит к исключению `IllegalArgumentException` внутри Source. Source возвращает ответ со статусом `500 Internal Server Error` и стектрейсом в теле.

Если мы обратимся к тому же эндпоинту через сервис RestClient, результат будет аналогичный — уродливый стектрейс вместо понятного сообщения об ошибке. Давайте это исправим.

## Подход в старом уроке (RestTemplate) vs. новый подход (RestClient)

В старом уроке для обработки ошибок нужно было:

1. Создать класс, реализующий интерфейс `ResponseErrorHandler`
2. Зарегистрировать его через `RestTemplateBuilder.errorHandler(...)`
3. Вся обработка — в одном месте, для всех запросов одинаковая

В `RestClient` подход гибче — обработка ошибок встроена прямо в цепочку вызовов через метод `.onStatus()`. Можно задать обработку глобально (при создании клиента) или точечно (для конкретного запроса). Отдельный класс `ResponseErrorHandler` больше не нужен.

## Код

### ErrorMessage

```java
package ru.job4j.restclient.model;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@AllArgsConstructor
public class ErrorMessage {
    private String message;
}
```

### AppException

```java
package ru.job4j.restclient.exception;

public class AppException extends RuntimeException {
    public AppException(String message) {
        super(message);
    }
}
```

### IdNotFoundException

```java
package ru.job4j.restclient.exception;

public class IdNotFoundException extends AppException {
    public IdNotFoundException(String message) {
        super(message);
    }
}
```

Эти три класса остались без изменений — иерархия исключений и модель ответа не зависят от HTTP-клиента.

### AppConfig — ключевое изменение

**Было (RestTemplate):**

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
                .errorHandler(new RestTemplateResponseErrorHandler())
                .build();
    }
}
```

**Стало (RestClient):**

```java
package ru.job4j.restclient.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatusCode;
import org.springframework.web.client.RestClient;
import ru.job4j.restclient.exception.IdNotFoundException;

@Configuration
public class AppConfig {

    @Value("${source-api-url}")
    private String url;

    @Bean
    public RestClient restClient() {
        return RestClient.builder()
                .baseUrl(url)
                .defaultStatusHandler(
                        HttpStatusCode::is5xxServerError,
                        (request, response) -> {
                            throw new IdNotFoundException("ID не найден");
                        }
                )
                .defaultStatusHandler(
                        HttpStatusCode::is4xxClientError,
                        (request, response) -> {
                            throw new IdNotFoundException("Ресурс не найден");
                        }
                )
                .build();
    }
}
```

Обратите внимание на разницу:

- Вместо отдельного класса `RestTemplateResponseErrorHandler` обработка ошибок задаётся прямо в builder'е через `.defaultStatusHandler()`.
- Метод принимает два аргумента: **предикат** (какие статус-коды перехватывать) и **обработчик** (что делать). Предикат — это `Predicate<HttpStatusCode>`, мы используем method reference `HttpStatusCode::is5xxServerError`.
- `defaultStatusHandler` применяется ко **всем** запросам, сделанным через этот `RestClient`. Это аналог глобального `ResponseErrorHandler` из `RestTemplate`.
- Можно зарегистрировать несколько обработчиков — они проверяются по порядку.

### Точечная обработка ошибок

Помимо глобальной обработки, `RestClient` позволяет задать обработку для конкретного запроса. Это полезно, когда разные эндпоинты требуют разной логики:

```java
public Item findById(int id) {
    return client.get()
            .uri("/getById?id={id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new IdNotFoundException(
                        "Item с id=%d не найден".formatted(id)
                );
            })
            .body(Item.class);
}
```

Здесь `.onStatus()` переопределяет глобальный обработчик только для этого запроса. Мы даже можем включить `id` в сообщение об ошибке — чего нельзя было сделать с глобальным `ResponseErrorHandler` в `RestTemplate`.

### ExceptionApiHandler — без изменений

```java
package ru.job4j.restclient.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import ru.job4j.restclient.exception.IdNotFoundException;
import ru.job4j.restclient.model.ErrorMessage;

@RestControllerAdvice
public class ExceptionApiHandler {

    @ExceptionHandler(IdNotFoundException.class)
    public ResponseEntity<ErrorMessage> notFoundException(IdNotFoundException exception) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorMessage(exception.getMessage()));
    }
}
```

`@RestControllerAdvice` — это механизм Spring MVC, он не привязан к конкретному HTTP-клиенту. Работает одинаково что с `RestTemplate`, что с `RestClient`.

## Как это работает

Цепочка обработки ошибки выглядит так:

```
1. RestClient отправляет запрос к Source
         │
         ▼
2. Source возвращает 500 (IllegalArgumentException)
         │
         ▼
3. defaultStatusHandler проверяет: is5xxServerError? → ДА
         │
         ▼
4. Обработчик выбрасывает IdNotFoundException("ID не найден")
         │
         ▼
5. ExceptionApiHandler перехватывает IdNotFoundException
         │
         ▼
6. Клиент получает 404 + {"message": "ID не найден"}
```

## Что удалено

Класс `RestTemplateResponseErrorHandler` **больше не нужен**. Его функциональность полностью заменена лямбдами в `.defaultStatusHandler()`. Это одно из преимуществ `RestClient` — меньше boilerplate-классов, логика обработки ошибок находится рядом с конфигурацией клиента, а не размазана по отдельным файлам.

## Итоговая структура сервиса RestClient

```
restclient/
├── RestClientApplication.java
├── config/
│   └── AppConfig.java                  ← RestClient + обработчики ошибок
├── controller/
│   ├── TrackerController.java
│   └── ExceptionApiHandler.java        ← @RestControllerAdvice
├── exception/
│   ├── AppException.java
│   └── IdNotFoundException.java
├── model/
│   ├── Item.java
│   └── ErrorMessage.java
└── service/
    └── TrackerService.java
```

## Сравнение подходов

| Критерий | RestTemplate (старый) | RestClient (новый) |
|----------|----------------------|-------------------|
| **Где обработка** | Отдельный класс `ResponseErrorHandler` | Лямбда в `.defaultStatusHandler()` |
| **Гранулярность** | Глобальная — одна на все запросы | Глобальная + точечная через `.onStatus()` |
| **Доступ к контексту запроса** | Нет — обработчик видит только ответ | Да — обработчик получает и `request`, и `response` |
| **Количество файлов** | +1 класс (обработчик) | 0 дополнительных файлов |
| **Тестируемость** | Нужно мокать `RestTemplate` | Встроенный `MockRestServiceServer` для `RestClient` |

## Ссылки

- [RestClient — Status Handlers](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-restclient-status-handlers) — документация Spring
- [Error Handling for REST with Spring](https://www.baeldung.com/exception-handling-for-rest-with-spring) — подробный разбор на Baeldung

## Задание

1. Добавьте в проект CheckDev обработку исключений с использованием `RestClient` и `.defaultStatusHandler()`.
2. *Дополнительно:* добавьте точечную обработку через `.onStatus()` для метода `findById`, чтобы сообщение об ошибке содержало конкретный `id`, который не был найден.
3. Создайте отдельную ветку для выполнения этой задачи. Оставьте ссылку на коммит.
4. Переведите ответственного на Арсентьева Петра.
