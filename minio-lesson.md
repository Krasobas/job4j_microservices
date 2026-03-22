# MinIO в Spring Boot: объектное хранилище от контейнера до кода

## Что такое MinIO и какую проблему решает

У тебя есть приложение, которое работает с файлами: фотографии пользователей, документы, отчёты. Где их хранить?

### Плохие варианты

**В файловой системе сервера** — работает ровно до тех пор, пока сервер один. Запустил второй экземпляр приложения — у него своя файловая система, файлов предыдущего он не видит. Сервер умер — файлы потеряны.

**В базе данных (BLOB)** — PostgreSQL поддерживает `bytea` и `Large Objects`. Но БД не оптимизирована для хранения файлов: бэкапы раздуваются, репликация замедляется, индексы страдают. JPEG на 5 МБ в таблице со 100 тысячами записей — это 500 ГБ в базе, которую ты бэкапишь каждый час.

### Правильный вариант: объектное хранилище

Объектное хранилище (Object Storage) — специализированная система для хранения неструктурированных данных. Каждый файл — это объект с ключом (именем), метаданными и телом. Объекты группируются в бакеты (buckets).

Amazon S3 — стандарт де-факто. Все остальные хранилища реализуют S3-совместимый API. Это означает: код, написанный для S3, работает с MinIO, Google Cloud Storage, Cloudflare R2 без изменений.

**MinIO** — open-source объектное хранилище с полной S3-совместимостью. Запускается одним бинарником или Docker-контейнером. В проде используется для приватных облаков, в разработке — как локальная замена Amazon S3. Ты разрабатываешь с MinIO локально, а в проде переключаешь endpoint на настоящий S3 — код менять не нужно.

---

## Ключевые концепции

**Bucket (бакет)** — контейнер для объектов. Аналог директории верхнего уровня, но без вложенных директорий в классическом понимании. Имя бакета глобально уникально (в рамках одного MinIO-сервера). Примеры: `students`, `avatars`, `documents`.

**Object (объект)** — файл внутри бакета. Идентифицируется ключом (key). Ключ может содержать `/`, имитируя директории: `photos/2024/avatar.jpg`. Но это не настоящие директории — это плоское пространство ключей.

**Key (ключ)** — уникальное имя объекта внутри бакета. В нашем проекте: `photo_001.jpg`, `photo_002.jpg`.

**Presigned URL** — временная ссылка на объект с встроенной аутентификацией. Можно отдать клиенту, и он скачает файл напрямую из MinIO без прокси через сервис. URL протухает через заданное время.

**Policy** — правила доступа к бакету. Можно сделать бакет публичным (download для всех), приватным (только авторизованные), или с точечными правилами на отдельные ключи.

---

## Шаг 1. MinIO в Docker Compose

### Контейнер MinIO

```yaml
services:
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  minio_data:
```

### Разбор по строкам

**`command: server /data --console-address ":9001"`** — MinIO запускается в серверном режиме. `/data` — директория для хранения объектов внутри контейнера. `--console-address ":9001"` — веб-консоль на отдельном порту. Без этого флага MinIO выберет случайный порт для консоли.

**Два порта:**
- `9000` — S3 API. Сюда приходят программные запросы (upload, download, list). Именно этот порт использует Java SDK.
- `9001` — Web Console. Браузерный интерфейс для управления: создание бакетов, загрузка файлов, настройка политик. Полезно для разработки и отладки.

**`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`** — credentials администратора. В проде используй длинные случайные значения и храни в секретах. Для разработки `minioadmin` / `minioadmin` — стандартная конвенция.

**`volumes: minio_data:/data`** — named volume для персистентности. Без него `docker compose down` удалит все загруженные файлы. С volume данные живут между рестартами.

**healthcheck** — `mc ready local` проверяет, что S3 API отвечает. `mc` (MinIO Client) встроен в образ. Health check нужен для `depends_on` с `condition: service_healthy` — другие контейнеры подождут, пока MinIO реально готов, а не просто стартовал процесс.

### Проверка после запуска

```bash
docker compose up -d minio
```

Открой `http://localhost:9001` в браузере. Логин: `minioadmin`, пароль: `minioadmin`. Ты увидишь пустой MinIO без бакетов.

---

## Шаг 2. Автоматическая инициализация

Для проекта нужен бакет `students` с тестовыми фотографиями. Создавать руками при каждом `docker compose up` — не вариант. Нужна автоматизация.

### Init-контейнер

```yaml
services:
  minio:
    # ... как выше

  minio-init:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    volumes:
      - ./init/minio/photos:/tmp/photos
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minioadmin minioadmin;
      mc mb --ignore-existing local/students;
      mc cp /tmp/photos/*.jpg local/students/;
      echo 'MinIO initialized: bucket created, photos uploaded';
      "
```

### Что здесь происходит

**`image: minio/mc`** — образ содержит только MinIO Client (`mc`), без сервера. Лёгкий, стартует мгновенно.

**`depends_on` с `condition: service_healthy`** — init-контейнер запускается только после того, как MinIO прошёл healthcheck. Без этого `mc` попытается подключиться к ещё не готовому серверу и упадёт.

**`mc alias set local http://minio:9000 minioadmin minioadmin`** — регистрирует MinIO-сервер под псевдонимом `local`. Все последующие команды используют этот псевдоним.

**`mc mb --ignore-existing local/students`** — создаёт бакет `students`. Флаг `--ignore-existing` делает команду идемпотентной: при повторном запуске не упадёт с ошибкой "bucket already exists".

**`mc cp /tmp/photos/*.jpg local/students/`** — копирует тестовые фотографии из примонтированной директории в бакет. Volume `./init/minio/photos:/tmp/photos` маппит локальную папку с тестовыми JPG-файлами.

**Init-контейнер отработал и завершился** — это нормально. `docker compose ps` покажет его в состоянии `Exited (0)`. Он выполнил свою работу и остановился.

### Подготовка тестовых фото

```bash
mkdir -p init/minio/photos
```

Положи туда три тестовых файла: `photo_001.jpg`, `photo_002.jpg`, `photo_003.jpg`. Для тестирования подойдут любые картинки, хоть пустые JPEG.

Если не хочешь хранить бинарные файлы в Git, можно генерировать заглушки прямо в init-контейнере:

```yaml
entrypoint: >
  /bin/sh -c "
  mc alias set local http://minio:9000 minioadmin minioadmin;
  mc mb --ignore-existing local/students;
  for i in 001 002 003; do
    echo 'placeholder' | mc pipe local/students/photo_$${i}.jpg;
  done;
  echo 'MinIO initialized';
  "
```

Но для реального проекта лучше положить настоящие фото через volume — так результат при тестировании виден в браузере.

---

## Шаг 3. Порты: когда пробрасывать, когда нет

В нашей архитектуре MinIO — **внутренний сервис**. Браузер пользователя не обращается к нему напрямую. Фото проксируется через Service R → Service S → MinIO.

Для production:

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
  # Никаких ports: — MinIO доступен только внутри Docker-сети
```

Для разработки пробрасываем порты, чтобы можно было зайти в веб-консоль и посмотреть содержимое бакетов. В compose.yaml можно использовать профили:

```yaml
minio:
  # ...
  ports:
    - "9000:9000"   # S3 API — для отладки через mc CLI с хоста
    - "9001:9001"   # Web Console — для браузера разработчика
```

В проде эти порты убираются. MinIO живёт только во внутренней Docker-сети, и к нему обращается только Service S.

---

## Шаг 4. Зависимость в Spring Boot

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.14</version>
</dependency>
```

MinIO Java SDK — это standalone-библиотека, не Spring-стартер. Spring Boot BOM **не управляет** её версией, поэтому `<version>` обязателен.

SDK транзитивно тянет OkHttp (HTTP-клиент) и несколько библиотек для XML-парсинга S3-ответов. Конфликтов со Spring Boot обычно нет.

### Почему MinIO SDK, а не AWS S3 SDK?

Оба варианта работают с MinIO. Но:

**MinIO SDK** — легче, заточен под MinIO. API чище: один Builder-паттерн, понятные имена методов. Не тянет за собой гигантские зависимости AWS.

**AWS S3 SDK v2** — тяжелее, но универсальнее. Если ты переключишься на настоящий S3 в проде, менять код не придётся. Зависимости существенно больше.

Для учебного проекта MinIO SDK — правильный выбор. Для продакшен-проекта, где MinIO может замениться на S3, стоит рассмотреть AWS SDK.

---

## Шаг 5. Конфигурация MinIO в Spring Boot

### application.yaml

```yaml
minio:
  url: ${MINIO_URL:http://localhost:9000}
  access-key: ${MINIO_ACCESS_KEY:minioadmin}
  secret-key: ${MINIO_SECRET_KEY:minioadmin}
  bucket: students
```

Обрати внимание на паттерн `${ENV_VAR:default}`. В Docker-окружении переменные придут из `environment` в compose.yaml. При локальной разработке используются дефолтные значения.

### MinioConfig.java

```java
package ru.job4j.services.config;

import io.minio.MinioClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MinioConfig {

    @Bean
    public MinioClient minioClient(
            @Value("${minio.url}") String url,
            @Value("${minio.access-key}") String accessKey,
            @Value("${minio.secret-key}") String secretKey) {
        return MinioClient.builder()
                .endpoint(url)
                .credentials(accessKey, secretKey)
                .build();
    }
}
```

### Почему через `@Configuration`, а не автоконфигурацию?

Для MinIO нет официального Spring Boot стартера. Есть community-проекты (spring-boot-starter-minio), но они часто отстают по версиям. Одна `@Configuration` с тремя строками — проще и надёжнее, чем тянуть стороннюю зависимость.

`MinioClient` — потокобезопасный. Один бин на всё приложение. Создаётся при старте контекста, переиспользуется всеми сервисами.

---

## Шаг 6. Проверка бакета при старте

Хорошая практика — проверить (и при необходимости создать) бакет при старте приложения, а не надеяться, что init-контейнер отработал:

### MinioInitializer.java

```java
package ru.job4j.services.config;

import io.minio.BucketExistsArgs;
import io.minio.MakeBucketArgs;
import io.minio.MinioClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class MinioInitializer implements CommandLineRunner {

    private static final Logger LOG = LoggerFactory.getLogger(MinioInitializer.class);

    private final MinioClient minioClient;
    private final String bucket;

    public MinioInitializer(MinioClient minioClient,
                            @Value("${minio.bucket}") String bucket) {
        this.minioClient = minioClient;
        this.bucket = bucket;
    }

    @Override
    public void run(String... args) throws Exception {
        boolean exists = minioClient.bucketExists(
                BucketExistsArgs.builder()
                        .bucket(bucket)
                        .build());
        if (!exists) {
            minioClient.makeBucket(
                    MakeBucketArgs.builder()
                            .bucket(bucket)
                            .build());
            LOG.info("Created bucket: {}", bucket);
        } else {
            LOG.info("Bucket already exists: {}", bucket);
        }
    }
}
```

`CommandLineRunner` выполняется после полной инициализации контекста Spring. Если MinIO недоступен — приложение упадёт при старте с понятной ошибкой, а не через 10 минут при первом запросе пользователя.

---

## Шаг 7. Сервис для работы с файлами

### PhotoService.java

```java
package ru.job4j.services.service;

import io.minio.*;
import io.minio.errors.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import java.io.InputStream;

@Service
public class PhotoService {

    private static final Logger LOG = LoggerFactory.getLogger(PhotoService.class);

    private final MinioClient minioClient;
    private final String bucket;

    public PhotoService(MinioClient minioClient,
                        @Value("${minio.bucket}") String bucket) {
        this.minioClient = minioClient;
        this.bucket = bucket;
    }

    /**
     * Download a photo by its key.
     * Returns byte array or empty array if not found.
     */
    public byte[] getPhoto(String photoKey) {
        try (InputStream stream = minioClient.getObject(
                GetObjectArgs.builder()
                        .bucket(bucket)
                        .object(photoKey)
                        .build())) {
            return stream.readAllBytes();
        } catch (ErrorResponseException e) {
            if ("NoSuchKey".equals(e.errorResponse().code())) {
                LOG.warn("Photo not found: {}", photoKey);
                return new byte[0];
            }
            throw new RuntimeException("MinIO error", e);
        } catch (Exception e) {
            throw new RuntimeException("Failed to get photo: " + photoKey, e);
        }
    }

    /**
     * Upload a photo.
     */
    public void uploadPhoto(String photoKey, InputStream stream,
                            long size, String contentType) {
        try {
            minioClient.putObject(
                    PutObjectArgs.builder()
                            .bucket(bucket)
                            .object(photoKey)
                            .stream(stream, size, -1)
                            .contentType(contentType)
                            .build());
            LOG.info("Uploaded photo: {}", photoKey);
        } catch (Exception e) {
            throw new RuntimeException("Failed to upload photo: " + photoKey, e);
        }
    }

    /**
     * Delete a photo.
     */
    public void deletePhoto(String photoKey) {
        try {
            minioClient.removeObject(
                    RemoveObjectArgs.builder()
                            .bucket(bucket)
                            .object(photoKey)
                            .build());
            LOG.info("Deleted photo: {}", photoKey);
        } catch (Exception e) {
            throw new RuntimeException("Failed to delete photo: " + photoKey, e);
        }
    }

    /**
     * Check if a photo exists.
     */
    public boolean exists(String photoKey) {
        try {
            minioClient.statObject(
                    StatObjectArgs.builder()
                            .bucket(bucket)
                            .object(photoKey)
                            .build());
            return true;
        } catch (ErrorResponseException e) {
            if ("NoSuchKey".equals(e.errorResponse().code())) {
                return false;
            }
            throw new RuntimeException("MinIO error", e);
        } catch (Exception e) {
            throw new RuntimeException("Failed to check photo: " + photoKey, e);
        }
    }
}
```

### Разбор API

**Builder-паттерн** — все операции MinIO SDK используют одну и ту же структуру: `XxxArgs.builder().bucket(...).object(...).build()`. Выглядит многословно, но у каждой операции свой набор параметров, и Builder делает это явным.

**`getObject()`** — возвращает `InputStream`. Важно закрыть его (`try-with-resources`), иначе HTTP-соединение с MinIO зависнет. `readAllBytes()` для маленьких файлов (фотографии) — нормально. Для больших (видео, архивы) нужен стриминг.

**`putObject()`** — принимает `InputStream`, размер и content type. Параметр `size` обязателен для потоков, где размер известен заранее. Если размер неизвестен, передаёшь `-1` как size и задаёшь `partSize` (минимум 5 МБ) — MinIO будет использовать multipart upload.

```java
// Когда размер неизвестен
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket(bucket)
        .object(key)
        .stream(stream, -1, 10 * 1024 * 1024) // partSize = 10MB
        .contentType(contentType)
        .build());
```

**`statObject()`** — аналог `HEAD` запроса в S3. Возвращает метаданные объекта (размер, content type, дату модификации) без скачивания тела. Используем для проверки существования.

**`removeObject()`** — удаление. Не бросает исключение, если объект не существует (идемпотентная операция в S3).

### Обработка ошибок

`ErrorResponseException` содержит S3-совместимый код ошибки. Основные:

| Код | Значение |
|---|---|
| `NoSuchKey` | Объект не найден |
| `NoSuchBucket` | Бакет не существует |
| `AccessDenied` | Нет прав |
| `InvalidBucketName` | Невалидное имя бакета |

Разделяй "объект не найден" (штатная ситуация, возвращаем 404) и "MinIO недоступен" (аварийная, пробрасываем RuntimeException). Не глотай все исключения в один catch.

---

## Шаг 8. HTTP-endpoint для раздачи фото

В нашей архитектуре фото проксируются через сервисы. MinIO не торчит наружу. Service S предоставляет внутренний endpoint для Service R:

### InternalPhotoController.java

```java
package ru.job4j.services.controller;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import ru.job4j.services.service.PhotoService;

@RestController
@RequestMapping("/internal/photos")
public class InternalPhotoController {

    private final PhotoService photoService;

    public InternalPhotoController(PhotoService photoService) {
        this.photoService = photoService;
    }

    @GetMapping("/{key}")
    public ResponseEntity<byte[]> getPhoto(@PathVariable String key) {
        if (!photoService.exists(key)) {
            return ResponseEntity.notFound().build();
        }
        byte[] photo = photoService.getPhoto(key);
        return ResponseEntity.ok()
                .header(HttpHeaders.CACHE_CONTROL, "max-age=3600")
                .contentType(resolveMediaType(key))
                .contentLength(photo.length)
                .body(photo);
    }

    private MediaType resolveMediaType(String key) {
        if (key.endsWith(".png")) {
            return MediaType.IMAGE_PNG;
        }
        if (key.endsWith(".gif")) {
            return MediaType.IMAGE_GIF;
        }
        return MediaType.IMAGE_JPEG;
    }
}
```

### Детали

**`Cache-Control: max-age=3600`** — фотографии меняются редко. Кэш на час снижает нагрузку на MinIO при повторных запросах. Service R тоже может кэшировать ответ.

**`contentLength`** — без него некоторые HTTP-клиенты не могут показать прогресс скачивания. Хорошая практика — всегда выставлять.

**Content type** определяем по расширению ключа. В реальном проекте лучше хранить content type в метаданных объекта MinIO или в БД.

**Почему `byte[]`, а не стриминг?** — для фотографий до 5-10 МБ `byte[]` достаточно. Для больших файлов нужен `StreamingResponseBody`:

```java
@GetMapping("/{key}")
public ResponseEntity<StreamingResponseBody> getPhotoStream(
        @PathVariable String key) {
    StreamingResponseBody body = outputStream -> {
        try (InputStream input = minioClient.getObject(
                GetObjectArgs.builder()
                        .bucket(bucket)
                        .object(key)
                        .build())) {
            input.transferTo(outputStream);
        }
    };
    return ResponseEntity.ok()
            .contentType(MediaType.IMAGE_JPEG)
            .body(body);
}
```

`StreamingResponseBody` передаёт данные chunk-by-chunk, не загружая весь файл в память. Для фото студентов overkill, но для крупных файлов — правильный подход.

---

## Шаг 9. Загрузка файлов (Upload)

В задании upload не требуется, но в реальном проекте он неизбежен. Покажу, как выглядит:

### Endpoint для загрузки

```java
@PostMapping(value = "/upload",
             consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<String> uploadPhoto(
        @RequestParam("file") MultipartFile file,
        @RequestParam("key") String key) {
    try {
        photoService.uploadPhoto(
                key,
                file.getInputStream(),
                file.getSize(),
                file.getContentType());
        return ResponseEntity.ok("Uploaded: " + key);
    } catch (Exception e) {
        return ResponseEntity.internalServerError()
                .body("Upload failed: " + e.getMessage());
    }
}
```

### Ограничение размера файла

В `application.yaml`:

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

Без этого Spring Boot по умолчанию ограничивает upload до 1 МБ. Для фотографий 10 МБ — разумный лимит.

---

## Шаг 10. Presigned URL — альтернативный подход

В нашем проекте мы проксируем фото через сервисы. Но есть более эффективный подход для случаев, когда MinIO доступен клиенту напрямую (например, через reverse proxy):

```java
@Service
public class PresignedUrlService {

    private final MinioClient minioClient;
    private final String bucket;

    public PresignedUrlService(MinioClient minioClient,
                               @Value("${minio.bucket}") String bucket) {
        this.minioClient = minioClient;
        this.bucket = bucket;
    }

    /**
     * Generate a temporary download URL.
     * The URL expires after the specified duration.
     */
    public String getDownloadUrl(String key, int expiryMinutes) {
        try {
            return minioClient.getPresignedObjectUrl(
                    GetPresignedObjectUrlArgs.builder()
                            .bucket(bucket)
                            .object(key)
                            .method(Method.GET)
                            .expiry(expiryMinutes, TimeUnit.MINUTES)
                            .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate presigned URL", e);
        }
    }

    /**
     * Generate a temporary upload URL.
     * Client can PUT a file directly to MinIO.
     */
    public String getUploadUrl(String key, int expiryMinutes) {
        try {
            return minioClient.getPresignedObjectUrl(
                    GetPresignedObjectUrlArgs.builder()
                            .bucket(bucket)
                            .object(key)
                            .method(Method.PUT)
                            .expiry(expiryMinutes, TimeUnit.MINUTES)
                            .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate presigned URL", e);
        }
    }
}
```

### Когда использовать presigned URL

**Прокси через сервис** (наш подход):
- MinIO скрыт во внутренней сети
- Сервис контролирует авторизацию
- Дополнительная нагрузка на сервис (трафик проходит через него)
- Простая архитектура

**Presigned URL** (прямой доступ):
- Клиент скачивает/загружает напрямую в MinIO
- Сервис не тратит ресурсы на проксирование трафика
- MinIO должен быть доступен клиенту (через reverse proxy или напрямую)
- Более сложная архитектура, но лучше масштабируется

Для учебного проекта прокси — правильный выбор. Для продакшена с большими файлами — presigned URL.

---

## Шаг 11. Actuator Health Check

Добавляем проверку доступности MinIO в Spring Boot Actuator:

### MinioHealthIndicator.java

```java
package ru.job4j.services.config;

import io.minio.BucketExistsArgs;
import io.minio.MinioClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class MinioHealthIndicator implements HealthIndicator {

    private final MinioClient minioClient;
    private final String bucket;

    public MinioHealthIndicator(MinioClient minioClient,
                                @Value("${minio.bucket}") String bucket) {
        this.minioClient = minioClient;
        this.bucket = bucket;
    }

    @Override
    public Health health() {
        try {
            boolean exists = minioClient.bucketExists(
                    BucketExistsArgs.builder()
                            .bucket(bucket)
                            .build());
            if (exists) {
                return Health.up()
                        .withDetail("bucket", bucket)
                        .build();
            }
            return Health.down()
                    .withDetail("bucket", bucket)
                    .withDetail("error", "bucket does not exist")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("bucket", bucket)
                    .withException(e)
                    .build();
        }
    }
}
```

Теперь `GET /actuator/health` покажет статус MinIO:

```json
{
  "status": "UP",
  "components": {
    "minio": {
      "status": "UP",
      "details": {
        "bucket": "students"
      }
    },
    "db": {
      "status": "UP"
    }
  }
}
```

В compose.yaml можно использовать это для health check сервиса:

```yaml
service-s:
  healthcheck:
    test: ["CMD-SHELL", "curl -f http://localhost:8081/actuator/health || exit 1"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---

## Шаг 12. Тестирование с Testcontainers

Для интеграционных тестов поднимаем реальный MinIO в контейнере:

### pom.xml (test scope)

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>minio</artifactId>
    <scope>test</scope>
</dependency>
```

### PhotoServiceIT.java

```java
@SpringBootTest
@Testcontainers
class PhotoServiceIT {

    @Container
    static MinIOContainer minio = new MinIOContainer("minio/minio")
            .withUserName("testuser")
            .withPassword("testpassword");

    @DynamicPropertySource
    static void minioProperties(DynamicPropertyRegistry registry) {
        registry.add("minio.url", minio::getS3URL);
        registry.add("minio.access-key", minio::getUserName);
        registry.add("minio.secret-key", minio::getPassword);
    }

    @Autowired
    private PhotoService photoService;

    @Test
    void uploadAndDownload() throws Exception {
        byte[] original = "test photo content".getBytes();
        var stream = new ByteArrayInputStream(original);

        photoService.uploadPhoto("test.jpg", stream,
                original.length, "image/jpeg");

        byte[] downloaded = photoService.getPhoto("test.jpg");
        assertArrayEquals(original, downloaded);
    }

    @Test
    void getNonExistentPhotoReturnsEmptyArray() {
        byte[] result = photoService.getPhoto("nonexistent.jpg");
        assertEquals(0, result.length);
    }

    @Test
    void existsReturnsFalseForMissingObject() {
        assertFalse(photoService.exists("missing.jpg"));
    }
}
```

`@DynamicPropertySource` подставляет URL и credentials реального MinIO-контейнера вместо значений из `application.yaml`. Тест работает с настоящим MinIO, а не с моками. Это даёт реальную уверенность, что код работает.

---

## Шаг 13. Безопасность

### Credentials

В compose.yaml для разработки `minioadmin/minioadmin` — нормально. Для прода:

```yaml
minio:
  environment:
    MINIO_ROOT_USER_FILE: /run/secrets/minio_user
    MINIO_ROOT_PASSWORD_FILE: /run/secrets/minio_password
  secrets:
    - minio_user
    - minio_password

secrets:
  minio_user:
    file: ./secrets/minio_user.txt
  minio_password:
    file: ./secrets/minio_password.txt
```

Docker Secrets монтируют файлы в `/run/secrets/`. MinIO читает credentials из файлов вместо переменных окружения. Переменные окружения видны в `docker inspect`, файлы — нет.

### Bucket policies

По умолчанию бакет приватный — доступ только с credentials. Если нужно открыть публичный доступ на чтение (например, для аватарок):

```bash
mc anonymous set download local/students
```

Или создать точечную политику через JSON:

```bash
mc anonymous set-json policy.json local/students
```

Для нашего проекта бакет приватный — доступ только через Service S с credentials.

---

## Полная картина: как фото доходит до браузера

```
Браузер
  │
  │  GET /api/students/RB-001/photo
  ▼
Service R (port 8080)
  │
  │  GET /internal/photos/photo_001.jpg
  │  (через RestClient, внутренняя Docker-сеть)
  ▼
Service S (port 8081)
  │
  │  InternalPhotoController → PhotoService → MinioClient
  │
  │  GetObjectArgs: bucket=students, object=photo_001.jpg
  │  (HTTP GET на http://minio:9000)
  ▼
MinIO (port 9000, внутренняя сеть)
  │
  │  Возвращает byte[] из volume minio_data
  ▼
Service S
  │
  │  ResponseEntity<byte[]> с Content-Type: image/jpeg
  ▼
Service R
  │
  │  Проксирует byte[] клиенту
  ▼
Браузер показывает фото
```

MinIO не торчит наружу. Авторизация на уровне Service R (Spring Security). Между сервисами — внутренняя Docker-сеть. Чисто и безопасно.

---

## Частые ошибки

### 1. Connection refused при старте

Service S стартует раньше MinIO. Решение — `depends_on` с `condition: service_healthy`:

```yaml
service-s:
  depends_on:
    minio:
      condition: service_healthy
```

### 2. Bucket does not exist

Init-контейнер не успел отработать, или упал с ошибкой. Решение — `MinioInitializer` в приложении (шаг 6) как страховка.

### 3. XML-парсинг ошибок OkHttp

MinIO SDK использует OkHttp, а Spring Boot может подтянуть свою версию. Конфликт версий проявляется как `NoSuchMethodError` в runtime. Решение — явно выровнять версию OkHttp:

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
```

Или позволить Spring Boot BOM управлять версией и убедиться, что MinIO SDK совместим.

### 4. Большие файлы убивают память

`readAllBytes()` грузит весь файл в heap. Для файла на 100 МБ — OOM. Для больших файлов используй `StreamingResponseBody` (шаг 8) или presigned URL (шаг 10).

### 5. Забыл content type при upload

Без `contentType()` MinIO сохранит файл с `application/octet-stream`. Браузер не распознает его как картинку и предложит скачать вместо отображения. Всегда передавай правильный MIME-type.
