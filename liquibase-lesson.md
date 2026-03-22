# Liquibase в Spring Boot: управление схемой базы данных

## Проблема, которую решает Liquibase

У тебя есть приложение с базой данных. Схема эволюционирует: добавляются таблицы, меняются колонки, заполняются справочники. Как управлять этими изменениями?

### Наивный подход: `ddl-auto=update`

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

Hibernate смотрит на JPA-сущности, сравнивает с текущей схемой БД и применяет изменения. Звучит удобно, но на практике это мина:

- Hibernate **не удаляет** колонки и таблицы. Переименовал поле в сущности — в БД появится новая колонка, старая останется с данными.
- Hibernate **не умеет** мигрировать данные. Нужно разбить одну таблицу на две? `ddl-auto` не поможет.
- Нет истории. Нельзя посмотреть, что и когда изменилось. Нельзя откатить.
- В проде `ddl-auto=update` — это русская рулетка. Один неосторожный деплой — и схема поехала, данные потеряны, откатить невозможно.

### Наивный подход #2: SQL-скрипты руками

Создаёшь файлы `V1_create_tables.sql`, `V2_add_column.sql`, запускаешь их руками или через CI. Лучше, но:

- Как понять, какие скрипты уже применены на конкретной БД?
- Как гарантировать порядок?
- Что если два разработчика создали конфликтующие скрипты?

### Решение: инструмент миграций

Liquibase (и его конкурент Flyway) решают именно это. Они хранят историю примененных изменений **в самой базе данных**, гарантируют порядок, обнаруживают конфликты и позволяют откатывать изменения.

---

## Liquibase vs Flyway

Оба инструмента живы и популярны в 2026. Ключевые отличия:

**Flyway** — проще. Миграции — это пронумерованные SQL-файлы (`V1__create_table.sql`, `V2__add_column.sql`). Минималистичный подход: пишешь SQL, Flyway применяет его по порядку. Если тебе хватает SQL и не нужна кроссплатформенность — Flyway проще в освоении.

**Liquibase** — мощнее. Поддерживает декларативные форматы (YAML, XML, JSON) помимо SQL. Может генерировать SQL под конкретную СУБД из одного описания. Имеет команду `diff` для сравнения баз. Поддерживает автоматический rollback для многих типов изменений.

Для этого урока мы используем Liquibase с **YAML-форматом** — он читаемее XML и нативнее для Spring Boot разработчика, который уже работает с `application.yaml`.

---

## Ключевые концепции

Прежде чем писать код, нужно понять четыре термина:

**Changelog** — файл, описывающий последовательность изменений. Это журнал миграций. Может быть один монолитный файл или master-файл, который ссылается на дочерние (рекомендуемый подход).

**Changeset** — атомарная единица изменения. Один changeset = одна операция: создать таблицу, добавить колонку, вставить данные. Каждый changeset имеет уникальный идентификатор (id + author) и выполняется ровно один раз.

**Change Type** — тип операции внутри changeset. `createTable`, `addColumn`, `insert`, `dropTable`, `sql` (для raw SQL). Liquibase знает, как преобразовать каждый тип в SQL для конкретной СУБД.

**DATABASECHANGELOG** — служебная таблица, которую Liquibase создаёт в твоей БД. В ней записывается каждый применённый changeset: id, author, дата, контрольная сумма. По этой таблице Liquibase понимает, что уже выполнено, а что ещё нет.

---

## Шаг 1. Зависимость

В `pom.xml`:

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

Версию указывать **не нужно** — Spring Boot BOM управляет ей. На момент Spring Boot 3.4.x подтягивается Liquibase 4.x. Если хочешь конкретную версию, переопредели в `<properties>`:

```xml
<properties>
    <liquibase.version>4.31.1</liquibase.version>
</properties>
```

Spring Boot автоматически запускает Liquibase при старте приложения, если `liquibase-core` есть в classpath. Никакой дополнительной конфигурации для базового сценария не нужно.

---

## Шаг 2. Убираем `ddl-auto`

Если в проекте был `ddl-auto=update`, меняем на `validate`:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

`validate` означает: Hibernate проверяет, что JPA-сущности соответствуют схеме БД, но **ничего не меняет**. Если схема не совпадает — приложение падает при старте с понятной ошибкой. Это страховка: ты видишь рассогласование до того, как оно попадёт в прод.

Создание и изменение схемы теперь полностью за Liquibase.

---

## Шаг 3. Структура файлов

```
src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog-master.yaml          # Master-файл
        └── migrations/
            ├── 001-create-students-table.yaml
            ├── 002-insert-test-data.yaml
            └── 003-add-email-column.yaml
```

**Почему именно `db/changelog/`?** — Spring Boot по умолчанию ищет master changelog в `classpath:db/changelog/db.changelog-master.yaml`. Это конвенция. Можно переопределить:

```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

Но лучше следовать конвенции — меньше конфигурации.

**Почему отдельные файлы для миграций?** — один монолитный changelog быстро превращается в нечитаемую портянку. Master-файл содержит только ссылки на дочерние, каждый дочерний — одна логическая миграция. При конфликте в Git конфликт будет в одну строку в master-файле (новый include), а не в середине тысячестрочного YAML.

**Нумерация с префиксом** (`001-`, `002-`) — гарантирует порядок при использовании `includeAll`. Liquibase применяет файлы в алфавитном порядке.

---

## Шаг 4. Master changelog

### db.changelog-master.yaml

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/migrations/001-create-students-table.yaml
  - include:
      file: db/changelog/migrations/002-insert-test-data.yaml
```

`include` загружает указанный файл и выполняет его changeset-ы. Порядок включений — это порядок применения миграций.

Альтернатива — `includeAll`:

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/migrations/
```

`includeAll` подхватывает все файлы из директории в алфавитном порядке. Удобно, но менее явно: новый разработчик не видит порядок, не прочитав имена файлов. Для больших проектов `include` безопаснее — ты контролируешь порядок явно.

---

## Шаг 5. Первая миграция — создание таблицы

### migrations/001-create-students-table.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-students-table
      author: vasily
      changes:
        - createTable:
            tableName: students
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: record_book_number
                  type: varchar(20)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: faculty
                  type: varchar(100)
                  constraints:
                    nullable: false
              - column:
                  name: last_name
                  type: varchar(100)
                  constraints:
                    nullable: false
              - column:
                  name: first_name
                  type: varchar(100)
                  constraints:
                    nullable: false
              - column:
                  name: middle_name
                  type: varchar(100)
              - column:
                  name: photo_key
                  type: varchar(255)
```

### Что здесь важно

**id** — уникальный идентификатор changeset-а. Комбинация `id + author + filepath` должна быть уникальной. Хорошая практика — использовать описательные имена (`001-create-students-table`), а не timestamp-ы (`20240115120000`). Описательное имя читается и в файловой системе, и в таблице DATABASECHANGELOG.

**author** — кто создал changeset. В продакшен-проектах это имя разработчика. Для учебного проекта можно использовать фиксированное значение.

**Один changeset = одно изменение** — это главная best practice. Не создавай две таблицы в одном changeset. Если откат потребуется для одной из них, ты не сможешь откатить частично. Атомарность changeset-а = атомарность отката.

**Типы данных** — Liquibase использует обобщённые типы. `bigint`, `varchar(100)`, `boolean` — Liquibase переведёт их в SQL, специфичный для конкретной СУБД. Для PostgreSQL `bigint` станет `BIGINT`, для MySQL тоже `BIGINT`, для Oracle — `NUMBER(19)`. Если нужен специфичный тип PostgreSQL (например, `jsonb`), используешь его напрямую — Liquibase передаст как есть, но потеряешь кроссплатформенность.

**constraints** — ограничения описываются декларативно. `nullable: false` → `NOT NULL`, `unique: true` → `UNIQUE`, `primaryKey: true` → `PRIMARY KEY`. Liquibase сам сгенерирует правильный DDL.

---

## Шаг 6. Вторая миграция — тестовые данные

### migrations/002-insert-test-data.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 002-insert-test-data
      author: vasily
      context: dev
      changes:
        - insert:
            tableName: students
            columns:
              - column:
                  name: record_book_number
                  value: "RB-001"
              - column:
                  name: faculty
                  value: "Computer Science"
              - column:
                  name: last_name
                  value: "Ivanov"
              - column:
                  name: first_name
                  value: "Ivan"
              - column:
                  name: middle_name
                  value: "Ivanovich"
              - column:
                  name: photo_key
                  value: "photo_001.jpg"
        - insert:
            tableName: students
            columns:
              - column:
                  name: record_book_number
                  value: "RB-002"
              - column:
                  name: faculty
                  value: "Mathematics"
              - column:
                  name: last_name
                  value: "Petrova"
              - column:
                  name: first_name
                  value: "Anna"
              - column:
                  name: middle_name
                  value: "Sergeevna"
              - column:
                  name: photo_key
                  value: "photo_002.jpg"
        - insert:
            tableName: students
            columns:
              - column:
                  name: record_book_number
                  value: "RB-003"
              - column:
                  name: faculty
                  value: "Physics"
              - column:
                  name: last_name
                  value: "Sidorov"
              - column:
                  name: first_name
                  value: "Dmitry"
              - column:
                  name: middle_name
                  value: "Alekseevich"
              - column:
                  name: photo_key
                  value: "photo_003.jpg"
```

### Context — разделение окружений

`context: dev` — этот changeset выполнится только если Liquibase запущен с контекстом `dev`. Тестовые данные не нужны в проде. В application.yaml:

```yaml
# application-dev.yaml
spring:
  liquibase:
    contexts: dev
```

```yaml
# application-prod.yaml
spring:
  liquibase:
    contexts: prod
```

Changeset без `context` выполняется всегда. Changeset с `context: dev` — только когда активен контекст `dev`. Это позволяет хранить тестовые данные рядом с миграциями схемы, не рискуя залить их в прод.

### Почему insert, а не loadData?

Для трёх записей `insert` нагляднее. Для десятков и сотен записей Liquibase поддерживает загрузку из CSV:

```yaml
- changeSet:
    id: 002-load-test-data
    author: vasily
    context: dev
    changes:
      - loadData:
          tableName: students
          file: db/changelog/data/students.csv
          separator: ","
          columns:
            - column:
                name: record_book_number
                type: STRING
            - column:
                name: faculty
                type: STRING
```

CSV-файл лежит рядом в `db/changelog/data/students.csv`. Для больших объёмов данных это чище.

---

## Шаг 7. Эволюция схемы — добавление колонки

Через месяц понадобилось хранить email студентов. Не трогаем `001-create-students-table.yaml` — он уже применён и зафиксирован в DATABASECHANGELOG. Создаём новый файл:

### migrations/003-add-email-column.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 003-add-email-column
      author: vasily
      changes:
        - addColumn:
            tableName: students
            columns:
              - column:
                  name: email
                  type: varchar(255)
                  constraints:
                    unique: true
```

И добавляем include в master:

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/migrations/001-create-students-table.yaml
  - include:
      file: db/changelog/migrations/002-insert-test-data.yaml
  - include:
      file: db/changelog/migrations/003-add-email-column.yaml
```

При следующем запуске приложения Liquibase увидит, что changeset-ы 001 и 002 уже в DATABASECHANGELOG, пропустит их и выполнит только 003.

### Золотое правило: никогда не редактируй применённые changeset-ы

Liquibase хранит контрольную сумму (checksum) каждого changeset-а. Если ты изменишь содержимое уже применённого changeset-а, Liquibase обнаружит несовпадение и откажется стартовать:

```
Validation Failed:
  1 changesets check sum was changed
```

Это защита. Если changeset уже применён на prod-базе, его изменение означает, что prod-схема отличается от того, что описано в коде. Единственный правильный путь — создать новый changeset с нужными изменениями.

---

## Шаг 8. Preconditions — защита от ошибок

Что если миграция запускается на базе, где таблица уже существует (например, перенос существующего проекта на Liquibase)?

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-students-table
      author: vasily
      preConditions:
        - onFail: MARK_RAN
        - not:
            - tableExists:
                tableName: students
      changes:
        - createTable:
            tableName: students
            columns:
              # ...
```

`preConditions` проверяет условие перед выполнением. `onFail: MARK_RAN` означает: если условие не выполнено (таблица уже есть), пометь changeset как выполненный, но не выполняй. Альтернативы:

- `HALT` — остановить выполнение (по умолчанию)
- `CONTINUE` — пропустить и не помечать как выполненный
- `WARN` — выдать предупреждение и продолжить
- `MARK_RAN` — пометить как выполненный без выполнения

Для миграции существующего проекта на Liquibase `MARK_RAN` — правильный выбор.

---

## Шаг 9. Rollback

### Автоматический rollback

Liquibase умеет автоматически генерировать rollback для некоторых change types:

| Change Type | Автоматический rollback |
|---|---|
| `createTable` | `dropTable` |
| `addColumn` | `dropColumn` |
| `createIndex` | `dropIndex` |
| `insert` | `delete` |
| `addForeignKeyConstraint` | `dropForeignKeyConstraint` |

Если ты используешь декларативные change types (а не raw SQL), rollback работает бесплатно.

### Явный rollback

Для сложных изменений или raw SQL автоматический rollback невозможен. Описываешь его вручную:

```yaml
databaseChangeLog:
  - changeSet:
      id: 004-rename-column
      author: vasily
      changes:
        - renameColumn:
            tableName: students
            oldColumnName: middle_name
            newColumnName: patronymic
            columnDataType: varchar(100)
      rollback:
        - renameColumn:
            tableName: students
            oldColumnName: patronymic
            newColumnName: middle_name
            columnDataType: varchar(100)
```

### Выполнение rollback

Из командной строки (через Maven-плагин или Liquibase CLI):

```bash
# Откатить последний changeset
liquibase rollback-count 1

# Откатить до конкретного тега
liquibase rollback --tag=v1.0
```

В Spring Boot Liquibase **не делает rollback автоматически**. Rollback — это осознанное действие, которое выполняет разработчик или CI/CD pipeline.

### Tagging для контрольных точек

```yaml
databaseChangeLog:
  - changeSet:
      id: tag-v1.0
      author: vasily
      changes:
        - tagDatabase:
            tag: v1.0
```

Тег позволяет откатить все changeset-ы, применённые после определённой точки.

---

## Шаг 10. Raw SQL — когда декларативного формата не хватает

Не всё описывается через change types. Создать partial index в PostgreSQL, настроить partitioning, вызвать хранимую процедуру — для этого нужен raw SQL:

```yaml
databaseChangeLog:
  - changeSet:
      id: 005-create-partial-index
      author: vasily
      changes:
        - sql:
            sql: >
              CREATE INDEX idx_students_active_faculty
              ON students (faculty)
              WHERE photo_key IS NOT NULL
      rollback:
        - sql:
            sql: DROP INDEX IF EXISTS idx_students_active_faculty
```

Или ссылка на отдельный SQL-файл:

```yaml
databaseChangeLog:
  - changeSet:
      id: 006-complex-migration
      author: vasily
      changes:
        - sqlFile:
            path: db/changelog/sql/006-complex-migration.sql
            splitStatements: true
            stripComments: true
      rollback:
        - sqlFile:
            path: db/changelog/sql/006-complex-migration-rollback.sql
```

При использовании raw SQL ты теряешь два преимущества: автоматический rollback и кроссплатформенность. Но получаешь полный контроль над SQL.

---

## Шаг 11. Конфигурация в application.yaml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/students_db
    username: app
    password: secret

  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false

  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
    enabled: true
    # contexts: dev         # Раскомментировать для dev-профиля
    # default-schema: public
    # drop-first: false     # НИКОГДА true в проде — удалит всё
```

### Что делает Spring Boot при старте

1. Создаёт `DataSource` из `spring.datasource.*`
2. Если `liquibase-core` в classpath и `spring.liquibase.enabled != false`:
   - Создаёт `SpringLiquibase` бин
   - Liquibase подключается к БД через тот же `DataSource`
   - Создаёт таблицы `DATABASECHANGELOG` и `DATABASECHANGELOGLOCK` (если их нет)
   - Читает master changelog
   - Для каждого changeset: проверяет, есть ли он в `DATABASECHANGELOG` → если нет, выполняет → записывает в `DATABASECHANGELOG`
3. После Liquibase стартует JPA/Hibernate
4. Если `ddl-auto=validate`, Hibernate проверяет соответствие сущностей и схемы

Порядок важен: Liquibase **всегда выполняется до Hibernate**. Spring Boot гарантирует это через зависимость бинов.

---

## Шаг 12. Работа с Docker и init-скриптом

В предыдущем уроке мы использовали `init.sql` для инициализации БД через Docker-volume:

```yaml
postgres:
  image: postgres:17
  volumes:
    - ./init/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

С Liquibase **этот файл больше не нужен**. Удаляем `init.sql` и volume-маппинг. Схема и данные теперь управляются Liquibase при старте Spring Boot:

```yaml
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

PostgreSQL создаёт пустую БД `students_db`. При старте Service S Liquibase подключается к ней и выполняет все миграции. Чисто и воспроизводимо.

---

## Шаг 13. Полная картина — что происходит при запуске

```
docker compose up
         │
         ▼
┌─────────────────┐
│  PostgreSQL      │  Создаёт пустую БД students_db
│  стартует        │  Healthcheck проходит
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Service S       │  Spring Boot стартует
│  стартует        │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│  Liquibase                                       │
│                                                  │
│  1. Подключается к students_db                   │
│  2. Создаёт DATABASECHANGELOG (пустая)           │
│  3. Создаёт DATABASECHANGELOGLOCK                │
│  4. Читает db.changelog-master.yaml              │
│  5. 001-create-students-table → ВЫПОЛНЯЕТ        │
│     → CREATE TABLE students (...)                │
│     → Записывает в DATABASECHANGELOG             │
│  6. 002-insert-test-data → ВЫПОЛНЯЕТ             │
│     → INSERT INTO students VALUES (...)          │
│     → Записывает в DATABASECHANGELOG             │
│  7. Все changeset-ы обработаны                   │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│  Hibernate                                       │
│                                                  │
│  ddl-auto=validate                               │
│  Сущность Student.java соответствует таблице     │
│  students? → ДА → Продолжаем                     │
└────────┬────────────────────────────────────────┘
         │
         ▼
   Приложение готово к работе
```

При повторном запуске Liquibase проверит DATABASECHANGELOG, увидит, что все changeset-ы уже применены, и ничего не сделает. Старт будет быстрым.

---

## Частые ошибки

### 1. Редактирование применённого changeset-а

```
Validation Failed: 1 changesets check sum was changed
```

Ты изменил содержимое changeset-а, который уже применён. Варианты:

- **Правильно**: создай новый changeset с нужными изменениями.
- **Для разработки**: удали запись из DATABASECHANGELOG и перезапусти. Но только на dev-базе.
- **Аварийно**: добавь `validCheckSum: ANY` в changeset (отключает проверку). Не используй в проде.

### 2. Конфликт порядка changeset-ов

Два разработчика одновременно создали `003-...yaml`. При мерже оба попадают в master changelog. Liquibase выполнит их в порядке включения, но если они зависят друг от друга (один создаёт колонку, второй её использует), порядок критичен. Решение — code review и коммуникация. Автоматизация не заменяет общение в команде.

### 3. `drop-first: true` в проде

```yaml
spring:
  liquibase:
    drop-first: true  # УДАЛЯЕТ ВСЮ БАЗУ ПЕРЕД МИГРАЦИЕЙ
```

Полезно для тестов. Смертельно для прода. Если используешь — только в test-профиле.

### 4. Слишком крупные changeset-ы

Один changeset, который создаёт 10 таблиц, 5 индексов и вставляет 1000 строк. Если на середине что-то падает, откатить нужно всё. Разбивай на атомарные changeset-ы — один changeset = одно логическое изменение.

### 5. Забыл обновить JPA-сущность после миграции

Liquibase добавил колонку `email` в таблицу, но `Student.java` не обновлён. С `ddl-auto=validate` приложение упадёт при старте — Hibernate обнаружит несоответствие. Это правильное поведение: лучше упасть рано с понятной ошибкой, чем работать с рассогласованной моделью.

---

## Рекомендации для продакшена

**Нумерация**: используй монотонно растущие префиксы (`001-`, `002-`). Для больших проектов — timestamp (`20260320-1200-`). Главное — порядок должен быть однозначным.

**Один changeset — одно изменение**: `createTable` отдельно, `createIndex` отдельно, `insert` отдельно. Легче откатывать, легче понимать историю.

**Всегда пиши rollback для raw SQL**: декларативные change types откатываются автоматически. SQL — нет. Без rollback-секции ты не сможешь откатить миграцию.

**Context для тестовых данных**: `context: dev` или `context: test`. Прод не должен содержать тестовых данных.

**Не используй `ddl-auto=update` вместе с Liquibase**: это два конкурирующих механизма управления схемой. Только `validate` или `none`.

**Храни changelog в Git**: миграции — это часть кода. Они версионируются, проходят code review, деплоятся вместе с приложением.

**Тестируй миграции**: в CI/CD pipeline поднимай чистую БД, применяй все миграции, запускай тесты. Testcontainers с PostgreSQL для этого идеален.

---

## Итоговая конфигурация для нашего проекта

### pom.xml (дополнение)

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

### application.yaml

```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/students_db}
    username: ${SPRING_DATASOURCE_USERNAME:app}
    password: ${SPRING_DATASOURCE_PASSWORD:secret}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

### Файлы миграций

```
src/main/resources/db/changelog/
├── db.changelog-master.yaml
└── migrations/
    ├── 001-create-students-table.yaml
    └── 002-insert-test-data.yaml
```

### compose.yaml (postgres без init.sql)

```yaml
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

Результат: при `docker compose up` PostgreSQL стартует с пустой базой, Service S поднимается, Liquibase создаёт таблицу и заливает тестовые данные, Hibernate валидирует схему. Всё управляется кодом, всё в Git, всё воспроизводимо.
