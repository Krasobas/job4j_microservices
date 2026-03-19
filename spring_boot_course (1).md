# 3.5. Spring Boot — полный курс (2026)

> **Стек**: Java 21, Spring Boot 3.4, Maven 3.9, PostgreSQL 17
>
> **Сквозной проект**: Job Tracker — REST API для управления задачами
>
> **Подход**: каждый урок начинается с проблемы, показывает грабли, объясняет выбор

---

## 3.5.0. Введение

### Проблема: почему «голый» Spring — это больно

Представьте: вы хотите написать REST API, который ходит в базу данных. На «голом» Spring Framework вам нужно:

1. Создать Maven-проект, руками подобрать 15–20 зависимостей (Spring MVC, Spring ORM, Hibernate, Jackson, Tomcat, пул соединений...) и молиться, что версии совместимы.
2. Написать XML или Java-конфигурацию для `DispatcherServlet`, `DataSource`, `EntityManagerFactory`, `TransactionManager`, `ViewResolver`...
3. Настроить Tomcat (скачать, развернуть WAR, настроить порт, контекст).
4. Написать код маппинга — `ObjectMapper`, `HttpMessageConverter`.
5. Начать, наконец, писать бизнес-логику.

Шаги 1–4 одинаковы в каждом проекте. Это **boilerplate** — бесполезный код, который не приносит ценности бизнесу.

Spring Boot убирает шаги 1–4 полностью.

### Что делает Spring Boot

Spring Boot — не замена Spring Framework. Это **надстройка**, которая:

**1. Стартеры** — готовые наборы зависимостей с гарантией совместимости.

Вместо:
```xml
<!-- 15 зависимостей, каждая со своей версией -->
<dependency>spring-webmvc 6.2.1</dependency>
<dependency>jackson-databind 2.18.2</dependency>
<dependency>tomcat-embed-core 10.1.34</dependency>
<dependency>hibernate-core 6.6.4</dependency>
<!-- ... и ещё 11 штук -->
```

Пишем:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Один стартер тянет всё нужное. Версии подобраны и протестированы вместе.

**2. Авто-конфигурация** — Spring Boot смотрит, что лежит в classpath, и настраивает всё сам.

Увидел `spring-boot-starter-web` → поднял Tomcat на порту 8080, настроил Jackson, зарегистрировал DispatcherServlet.

Увидел `postgresql` в зависимостях + `spring.datasource.url` в конфигурации → создал `DataSource`, `EntityManagerFactory`, `TransactionManager`.

Вам не нужно писать ни строчки конфигурации для стандартных сценариев.

**3. Встроенный сервер** — Tomcat вшит в приложение. Запуск: `java -jar app.jar`. Не нужен внешний сервер.

**4. Convention over Configuration** — если вы не настроили что-то явно, Spring Boot применит разумный дефолт.

### Создаём проект

Идём на https://start.spring.io:

- **Project**: Maven
- **Language**: Java
- **Spring Boot**: 3.4.x (выбирайте последний стабильный)
- **Java**: 21
- **Group**: ru.job4j
- **Artifact**: tracker
- **Dependencies**: Spring Web

Нажимаем Generate → скачиваем ZIP → распаковываем → открываем в IDE.

Или в IntelliJ: File → New → Project → Spring Boot (Spring Initializr встроен).

### Что внутри

```
tracker/
├── src/
│   ├── main/
│   │   ├── java/ru/job4j/tracker/
│   │   │   └── TrackerApplication.java       ← точка входа
│   │   └── resources/
│   │       └── application.properties         ← конфигурация
│   └── test/
│       └── java/ru/job4j/tracker/
│           └── TrackerApplicationTests.java
├── pom.xml
└── mvnw / mvnw.cmd                           ← Maven Wrapper
```

**TrackerApplication.java**:

```java
@SpringBootApplication
public class TrackerApplication {
    public static void main(String[] args) {
        SpringApplication.run(TrackerApplication.class, args);
    }
}
```

Одна аннотация, одна строка кода — и у вас работающий веб-сервер.

`@SpringBootApplication` — это три аннотации в одной:
- `@Configuration` — этот класс является источником бинов.
- `@EnableAutoConfiguration` — включает авто-конфигурацию.
- `@ComponentScan` — сканирует пакет и вложенные на `@Component`, `@Service`, `@Repository`, `@Controller`.

### application.yml vs application.properties

Первое, что делаю в новом проекте — переименовываю `application.properties` в `application.yml`. YAML компактнее для вложенных свойств:

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: tracker
```

vs.

```properties
# application.properties
server.port=8080
spring.application.name=tracker
```

Для одного уровня — без разницы. Для трёх — YAML читабельнее. Выбирайте что нравится, но будьте последовательны в проекте.

### Первый контроллер

Создадим файл `controller/IndexController.java`:

```java
@RestController
public class IndexController {

    @GetMapping("/")
    public Map<String, String> index() {
        return Map.of(
            "service", "Job Tracker",
            "status", "running"
        );
    }
}
```

Запускаем: `mvn spring-boot:run` или зелёная стрелка в IDE.

Открываем http://localhost:8080:

```json
{"service":"Job Tracker","status":"running"}
```

Мы не настраивали Tomcat, Jackson, маршрутизацию. Spring Boot сделал всё сам.

### Профили: разные настройки для разных сред

На разработке вы подключаетесь к локальной БД. В production — к другой. Жёстко прописать URL в одном файле — не вариант.

**Решение: профили.**

```yaml
# application.yml — общее для всех
spring:
  application:
    name: tracker
  jpa:
    hibernate:
      ddl-auto: validate
```

```yaml
# application-dev.yml — для разработки
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/tracker
    username: postgres
    password: password
```

```yaml
# application-prod.yml — для production
server:
  port: 8080
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
```

Активация:
- В IDE: `--spring.profiles.active=dev`
- В Docker: `SPRING_PROFILES_ACTIVE=prod`
- В application.yml: `spring.profiles.active: dev`

Приоритет (от высшего к низшему):
1. Аргументы командной строки (`--server.port=9090`)
2. Переменные окружения (`SERVER_PORT=9090`)
3. `application-{profile}.yml`
4. `application.yml`

Это значит: вы задали порт 8080 в `application.yml`, но передали переменную `SERVER_PORT=9090` — приложение запустится на 9090.

### Структура пакетов

Вопрос, который возникает сразу: как организовать код?

Есть два подхода:

**По слоям (layer-first)** — классический:

```
ru.job4j.tracker/
├── controller/
│   ├── ItemController.java
│   └── UserController.java
├── service/
│   ├── ItemService.java
│   └── UserService.java
├── repository/
│   ├── ItemRepository.java
│   └── UserRepository.java
├── model/
│   ├── Item.java
│   └── User.java
└── dto/
    ├── ItemRequest.java
    └── ItemResponse.java
```

**По фичам (feature-first)** — для крупных проектов:

```
ru.job4j.tracker/
├── item/
│   ├── ItemController.java
│   ├── ItemService.java
│   ├── ItemRepository.java
│   ├── Item.java
│   └── dto/
├── user/
│   ├── UserController.java
│   ├── UserService.java
│   └── ...
└── config/
```

Для учебного проекта — **по слоям**. Он проще и привычнее. Для проекта с 50+ сущностями — по фичам, иначе в каждом пакете будет по 50 файлов.

### Грабли новичков

**1. Пакет не сканируется.** `@SpringBootApplication` сканирует свой пакет и вложенные. Если `TrackerApplication` в `ru.job4j.tracker`, а контроллер в `ru.job4j.controller` — он не найдётся. Контроллер должен быть в `ru.job4j.tracker` или глубже.

**2. Зависимость добавлена, но ничего не работает.** Авто-конфигурация срабатывает при наличии зависимости в classpath **И** необходимых свойств. Если добавили `spring-boot-starter-data-jpa`, но не указали `spring.datasource.url` — приложение упадёт при старте.

**3. Два стартера конфликтуют.** `spring-boot-starter-web` (Tomcat + Spring MVC) и `spring-boot-starter-webflux` (Netty + WebFlux) вместе — Spring запутается. Выберите один.

### Задание

1. Создайте проект через Spring Initializr (Spring Web, Java 21, Spring Boot 3.4).
2. Переименуйте `application.properties` в `application.yml`.
3. Создайте контроллер, возвращающий JSON на GET `/`.
4. Создайте профили `dev` и `prod` с разными портами (8080 и 9090). Переключитесь — убедитесь, что порт меняется.
5. Запустите с флагом `--debug` — изучите лог авто-конфигурации. Найдите строку, где Spring Boot создаёт Tomcat.

---

## 3.5.1. Data

### Проблема: как работать с базой данных без боли

Допустим, вам нужно сохранить задачу (Item) в PostgreSQL. На «голом» JDBC:

```java
// 20 строк на один INSERT
String sql = "INSERT INTO items(name, description, created_at) VALUES (?, ?, ?)";
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, item.getName());
    ps.setString(2, item.getDescription());
    ps.setTimestamp(3, Timestamp.valueOf(item.getCreatedAt()));
    ps.executeUpdate();
    try (ResultSet rs = ps.getGeneratedKeys()) {
        if (rs.next()) {
            item.setId(rs.getLong(1));
        }
    }
}
// А теперь напишите такой код для findAll, findById, update, delete...
// А потом для User, Comment, Category...
```

Каждая операция — 15–20 строк шаблонного кода. Для 10 сущностей это 500+ строк boilerplate.

Spring Data JPA сводит это к:

```java
public interface ItemRepository extends JpaRepository<Item, Long> { }
```

Ноль строк реализации. `save`, `findAll`, `findById`, `delete` — уже работают.

### Подключение

Добавьте зависимости в `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Конфигурация:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/tracker
    username: postgres
    password: password
  jpa:
    hibernate:
      ddl-auto: validate     # НИКОГДА не create/update в production
    show-sql: true            # полезно при разработке, выключить в production
    properties:
      hibernate:
        format_sql: true
```

### Entity: маппинг Java-класса на таблицу

```java
@Entity
@Table(name = "items")
public class Item {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Status status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() {
        this.createdAt = LocalDateTime.now();
        if (this.status == null) {
            this.status = Status.NEW;
        }
    }

    // конструкторы, геттеры, сеттеры, equals/hashCode по id
}
```

```java
public enum Status {
    NEW, IN_PROGRESS, DONE, CANCELLED
}
```

Разберём каждую аннотацию — **не запоминайте, поймите зачем**:

- `@Entity` — «этот класс маппится на таблицу». Без неё Hibernate не знает о классе.
- `@Table(name = "items")` — явное имя таблицы. Без неё Hibernate возьмёт имя класса (`Item` → таблица `item`). Я всегда указываю явно — меньше сюрпризов.
- `@Id` — первичный ключ.
- `@GeneratedValue(strategy = GenerationType.IDENTITY)` — БД сама генерирует ID (`SERIAL` / `BIGSERIAL` в PostgreSQL).
- `@Column(nullable = false)` — NOT NULL в таблице. Hibernate валидирует при сохранении. Это **не замена** constraint'у в миграции — это подсказка Hibernate.
- `@Enumerated(EnumType.STRING)` — сохраняет enum как строку (`'NEW'`), а не число. **Всегда используйте STRING.** Если использовать `ORDINAL` (по умолчанию), то `NEW` = 0, `IN_PROGRESS` = 1. Добавили новый статус в середину enum — все данные поехали.
- `@ManyToOne(fetch = FetchType.LAZY)` — связь «много задач → один пользователь». **LAZY обязателен.** Подробнее ниже.
- `@JoinColumn(name = "user_id")` — имя FK-колонки.
- `@PrePersist` — метод, который вызывается перед первым сохранением. Удобно для `createdAt`.

> **Важно**: с Spring Boot 3.0 пакет `javax.persistence` заменён на `jakarta.persistence`. Если IDE подсказывает `javax` — это старая версия.

### Миграции: Liquibase

**Никогда не используйте `ddl-auto: create` или `update` в production.**

Почему `update` опасен:
- Он умеет добавлять колонки, но **не умеет** удалять и переименовывать.
- Он не удаляет таблицы, которые вы убрали из кода.
- Он не заполняет данные по умолчанию.
- Он не работает в команде: два разработчика изменили Entity по-разному — конфликт.
- В production вы не контролируете, что именно произойдёт с данными.

**Решение**: миграции через Liquibase (или Flyway).

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
  jpa:
    hibernate:
      ddl-auto: validate   # только ПРОВЕРЯЕТ, что Entity соответствует таблице
```

**db/changelog/db.changelog-master.xml**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <include file="db/changelog/001-create-users-table.xml"/>
    <include file="db/changelog/002-create-items-table.xml"/>
</databaseChangeLog>
```

**db/changelog/001-create-users-table.xml**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                       http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="001" author="developer">
        <createTable tableName="users">
            <column name="id" type="BIGSERIAL">
                <constraints primaryKey="true"/>
            </column>
            <column name="username" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="role" type="VARCHAR(50)">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

**db/changelog/002-create-items-table.xml**:

```xml
<changeSet id="002" author="developer">
    <createTable tableName="items">
        <column name="id" type="BIGSERIAL">
            <constraints primaryKey="true"/>
        </column>
        <column name="name" type="VARCHAR(255)">
            <constraints nullable="false"/>
        </column>
        <column name="description" type="TEXT"/>
        <column name="status" type="VARCHAR(50)">
            <constraints nullable="false"/>
        </column>
        <column name="user_id" type="BIGINT">
            <constraints nullable="false"
                         foreignKeyName="fk_items_user"
                         references="users(id)"/>
        </column>
        <column name="created_at" type="TIMESTAMP">
            <constraints nullable="false"/>
        </column>
    </createTable>
</changeSet>
```

При запуске Liquibase:
1. Создаёт таблицу `databasechangelog`, где хранит историю миграций.
2. Проверяет, какие changeSet'ы уже применены.
3. Применяет только новые.

Добавили колонку? Создайте новый changeSet (003-add-priority-column.xml). Откатить? Liquibase умеет rollback.

### Repository

```java
public interface ItemRepository extends JpaRepository<Item, Long> {
    // Всё. findAll, findById, save, delete, count, existsById — уже работают.
}
```

`JpaRepository<Item, Long>` — дженерики: тип Entity и тип ID.

Встроенные методы:

```java
// Создание / обновление
Item saved = itemRepository.save(item);           // INSERT или UPDATE
List<Item> saved = itemRepository.saveAll(items);  // batch

// Чтение
List<Item> all = itemRepository.findAll();
Optional<Item> byId = itemRepository.findById(1L);
List<Item> byIds = itemRepository.findAllById(List.of(1L, 2L, 3L));
long count = itemRepository.count();
boolean exists = itemRepository.existsById(1L);

// Удаление
itemRepository.deleteById(1L);
itemRepository.delete(item);
itemRepository.deleteAll();
```

### Кастомные запросы: магия имён методов

Spring Data генерирует SQL из имени метода:

```java
public interface ItemRepository extends JpaRepository<Item, Long> {

    // WHERE name = ?
    Optional<Item> findByName(String name);

    // WHERE status = ?
    List<Item> findByStatus(Status status);

    // WHERE name LIKE %?%  (регистронезависимо)
    List<Item> findByNameContainingIgnoreCase(String search);

    // WHERE status = ? AND user_id = ?
    List<Item> findByStatusAndUser(Status status, User user);

    // WHERE created_at > ?  ORDER BY created_at DESC
    List<Item> findByCreatedAtAfterOrderByCreatedAtDesc(LocalDateTime after);

    // COUNT WHERE status = ?
    long countByStatus(Status status);

    // EXISTS WHERE name = ?
    boolean existsByName(String name);

    // DELETE WHERE status = ?
    void deleteByStatus(Status status);
}
```

Правило: `findBy` + имя поля + условие. Spring разбирает имя метода и генерирует JPQL.

Доступные ключевые слова: `And`, `Or`, `Between`, `LessThan`, `GreaterThan`, `Like`, `Containing`, `In`, `OrderBy`, `Not`, `True`, `False`, `IsNull`, `IsNotNull`.

**Когда имена методов становятся нечитаемыми** (больше 2–3 условий), используйте `@Query`:

```java
@Query("SELECT i FROM Item i WHERE i.status = :status AND i.user.id = :userId ORDER BY i.createdAt DESC")
List<Item> findUserItemsByStatus(@Param("status") Status status, @Param("userId") Long userId);

// Native SQL (когда JPQL не хватает)
@Query(value = "SELECT * FROM items WHERE name ILIKE '%' || :search || '%'", nativeQuery = true)
List<Item> search(@Param("search") String search);
```

### Пагинация

Отдавать 10 000 записей одним запросом — плохая идея. Пагинация — must have.

```java
// В репозитории: метод уже есть в JpaRepository
// Page<Item> findAll(Pageable pageable);

// В сервисе
public Page<Item> findAll(int page, int size) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
    return itemRepository.findAll(pageable);
}
```

`Page<Item>` содержит:
- `content` — список элементов на текущей странице
- `totalElements` — общее количество
- `totalPages` — общее количество страниц
- `number` — номер текущей страницы (с 0)
- `size` — размер страницы

Пагинация в кастомных запросах:

```java
Page<Item> findByStatus(Status status, Pageable pageable);
```

### N+1 проблема — самая частая ошибка

Это проблема, которая **не видна в коде**, но **убивает производительность**.

```java
@Entity
public class Item {
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
}
```

Кажется, всё правильно — LAZY загрузка. Но:

```java
List<Item> items = itemRepository.findAll();  // 1 SQL-запрос: SELECT * FROM items
for (Item item : items) {
    System.out.println(item.getUser().getUsername());
    // ↑ Для КАЖДОГО item отдельный запрос: SELECT * FROM users WHERE id = ?
}
```

Если items = 100, то выполнится **101 SQL-запрос** (1 на items + 100 на users). Это N+1.

**Решение 1: JOIN FETCH**

```java
@Query("SELECT i FROM Item i JOIN FETCH i.user")
List<Item> findAllWithUser();
```

Один SQL-запрос с JOIN. Все данные загружены сразу.

**Решение 2: @EntityGraph**

```java
@EntityGraph(attributePaths = {"user"})
List<Item> findAll();
```

То же самое, но декларативно. Удобно, когда не хотите писать JPQL.

**Как обнаружить N+1?**

Включите логирование SQL:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # показывает параметры запросов
```

Увидели десятки одинаковых `SELECT ... FROM users WHERE id = ?` — это N+1.

### Связи между Entity

**@ManyToOne** — самая частая связь. Много задач → один пользователь.

```java
@Entity
public class Item {
    @ManyToOne(fetch = FetchType.LAZY)        // ВСЕГДА LAZY
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
}
```

> Дефолт для `@ManyToOne` — **EAGER**. Это значит: при загрузке Item всегда загружается User. Даже если вам User не нужен. Это потенциальный N+1. **Всегда ставьте `LAZY` явно.**

**@OneToMany** — обратная сторона. Один пользователь → много задач.

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")   // "user" — имя поля в Item
    private List<Item> items;       // не загружается, пока не обратимся
}
```

> `@OneToMany` по умолчанию LAZY — тут дефолт правильный.

**@ManyToMany** — многие ко многим (например, теги).

```java
@Entity
public class Item {
    @ManyToMany
    @JoinTable(
        name = "item_tags",
        joinColumns = @JoinColumn(name = "item_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();
}
```

### equals/hashCode для Entity

Частый вопрос: как реализовать `equals`/`hashCode` для JPA-сущностей?

**Правило: по бизнес-ключу или по id (если не null).**

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Item item = (Item) o;
    return id != null && id.equals(item.id);
}

@Override
public int hashCode() {
    return getClass().hashCode();   // фиксированное значение — безопасно для коллекций
}
```

Почему не `Objects.hash(id)`? Потому что до `save()` id = null. Если объект попал в `HashSet` до save, а потом id изменился — объект «потеряется» в коллекции.

### Тестирование Data-слоя

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class ItemRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:17");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ItemRepository itemRepository;

    @Autowired
    private UserRepository userRepository;

    @Test
    void whenSaveItem_thenCanFindById() {
        User user = userRepository.save(new User("test", "pass", "ROLE_USER"));
        Item item = new Item("Fix bug", "Critical bug in login", user);
        Item saved = itemRepository.save(item);

        Optional<Item> found = itemRepository.findById(saved.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Fix bug");
    }

    @Test
    void whenFindByStatus_thenReturnsMatching() {
        User user = userRepository.save(new User("test", "pass", "ROLE_USER"));
        itemRepository.save(new Item("Task 1", null, Status.NEW, user));
        itemRepository.save(new Item("Task 2", null, Status.DONE, user));
        itemRepository.save(new Item("Task 3", null, Status.NEW, user));

        List<Item> newItems = itemRepository.findByStatus(Status.NEW);

        assertThat(newItems).hasSize(2);
    }
}
```

`@DataJpaTest` поднимает **только** JPA-слой: Entity, Repository, Hibernate, Liquibase. Контроллеры, Security, другие сервисы — не загружаются. Тест быстрый.

`Testcontainers` поднимает реальный PostgreSQL в Docker. Тест проверяет реальное поведение БД, а не in-memory заглушку.

Зависимости для тестов:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### Задание

1. Создайте Entity: `User` (id, username, password, role, createdAt) и `Item` (id, name, description, status, user, createdAt).
2. Создайте Liquibase-миграции для обеих таблиц.
3. Создайте `ItemRepository` с методами: `findByStatus`, `findByNameContainingIgnoreCase`, `findByStatusAndUser`.
4. Добавьте пагинацию: `findAll(Pageable)`.
5. Включите логирование SQL. Выполните `findAll()` + обращение к `item.getUser()` в цикле. Обнаружьте N+1 проблему.
6. Решите N+1 через `@Query` с `JOIN FETCH`.
7. Напишите тест с Testcontainers.
## 3.5.2. REST API

### Проблема: как правильно спроектировать API

REST API — это контракт между вами и клиентом (фронтенд, мобильное приложение, другой сервис). Плохой контракт = боль при каждом изменении. Хороший = годы без переписывания.

В этом уроке мы построим полноценный API для Job Tracker и разберём типичные ошибки, которые делают API неудобным.

### Слой за слоем: Controller → Service → Repository

```
HTTP-запрос
     │
     ▼
┌────────────┐  Принимает HTTP, валидирует, отдаёт HTTP.
│ Controller │  Не содержит бизнес-логики. Тонкий.
└─────┬──────┘
      │ DTO
      ▼
┌────────────┐  Бизнес-логика. Проверки, расчёты, оркестрация.
│  Service   │  Не знает про HTTP.
└─────┬──────┘
      │ Entity
      ▼
┌────────────┐  Работа с данными. SQL/JPQL.
│ Repository │  Не знает про бизнес-логику.
└────────────┘
```

**Почему не писать всё в контроллере?**

1. Контроллер привязан к HTTP. Если завтра нужно вызвать ту же логику из Kafka-listener'а — придётся дублировать.
2. Тестировать контроллер сложнее (MockMvc, HTTP-статусы). Тестировать сервис — просто unit-тест.
3. В контроллере нет транзакций. `@Transactional` на контроллере — антипаттерн.

### Контроллер

```java
@RestController
@RequestMapping("/api/items")
@RequiredArgsConstructor
public class ItemController {

    private final ItemService itemService;

    @GetMapping
    public Page<ItemResponse> getAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return itemService.findAll(page, size);
    }

    @GetMapping("/{id}")
    public ItemResponse getOne(@PathVariable Long id) {
        return itemService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ItemResponse create(@Valid @RequestBody CreateItemRequest request) {
        return itemService.create(request);
    }

    @PutMapping("/{id}")
    public ItemResponse update(@PathVariable Long id,
                               @Valid @RequestBody UpdateItemRequest request) {
        return itemService.update(id, request);
    }

    @PatchMapping("/{id}/status")
    public ItemResponse changeStatus(@PathVariable Long id,
                                     @RequestBody ChangeStatusRequest request) {
        return itemService.changeStatus(id, request.status());
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        itemService.delete(id);
    }
}
```

Обратите внимание:
- Контроллер **тонкий** — делегирует всё сервису.
- Возвращает **DTO** (`ItemResponse`), не Entity.
- `@Valid` включает валидацию.
- `@ResponseStatus(HttpStatus.CREATED)` — 201 для POST, 204 для DELETE.

### DTO: почему не Entity?

```java
// ❌ ПЛОХО: Entity наружу
@GetMapping("/{id}")
public Item getOne(@PathVariable Long id) {
    return itemRepository.findById(id).orElseThrow();
}
```

Проблемы:
1. **Утечка внутренней структуры.** Клиент видит поле `password` у User.
2. **Циклическая сериализация.** Item → User → List<Item> → User → ... → StackOverflow.
3. **Связанность.** Переименовали поле в Entity — сломали API.
4. **Лишние данные.** Для списка задач не нужен `description` (он длинный), а в детальном view — нужен.

**Решение: DTO (Data Transfer Object).**

```java
// Что приходит от клиента при создании
public record CreateItemRequest(
        @NotBlank(message = "Name is required")
        @Size(max = 255, message = "Name must be at most 255 characters")
        String name,

        String description
) {}

// Что приходит при обновлении (может отличаться от создания)
public record UpdateItemRequest(
        @NotBlank String name,
        String description
) {}

// Что уходит клиенту
public record ItemResponse(
        Long id,
        String name,
        String description,
        String status,
        String username,      // ← не весь User, а только имя
        LocalDateTime createdAt
) {}

// Для смены статуса
public record ChangeStatusRequest(
        @NotNull Status status
) {}
```

Java Records (Java 16+) идеальны для DTO: неизменяемые, компактные, `equals`/`hashCode`/`toString` из коробки.

### Service: бизнес-логика

```java
@Service
@RequiredArgsConstructor
public class ItemService {

    private final ItemRepository itemRepository;
    private final UserRepository userRepository;

    public Page<ItemResponse> findAll(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return itemRepository.findAllWithUser(pageable)
                .map(this::toResponse);
    }

    public ItemResponse findById(Long id) {
        Item item = getItemOrThrow(id);
        return toResponse(item);
    }

    @Transactional
    public ItemResponse create(CreateItemRequest request) {
        // TODO: получать userId из SecurityContext (после урока Security)
        User user = userRepository.findById(1L)
                .orElseThrow(() -> new ResourceNotFoundException("User", 1L));
        Item item = new Item();
        item.setName(request.name());
        item.setDescription(request.description());
        item.setUser(user);
        Item saved = itemRepository.save(item);
        return toResponse(saved);
    }

    @Transactional
    public ItemResponse update(Long id, UpdateItemRequest request) {
        Item item = getItemOrThrow(id);
        item.setName(request.name());
        item.setDescription(request.description());
        Item saved = itemRepository.save(item);
        return toResponse(saved);
    }

    @Transactional
    public ItemResponse changeStatus(Long id, Status newStatus) {
        Item item = getItemOrThrow(id);
        item.setStatus(newStatus);
        return toResponse(itemRepository.save(item));
    }

    @Transactional
    public void delete(Long id) {
        if (!itemRepository.existsById(id)) {
            throw new ResourceNotFoundException("Item", id);
        }
        itemRepository.deleteById(id);
    }

    private Item getItemOrThrow(Long id) {
        return itemRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Item", id));
    }

    private ItemResponse toResponse(Item item) {
        return new ItemResponse(
                item.getId(),
                item.getName(),
                item.getDescription(),
                item.getStatus().name(),
                item.getUser().getUsername(),
                item.getCreatedAt()
        );
    }
}
```

`@Transactional` — на методах, которые изменяют данные. Если внутри метода произойдёт исключение — все изменения откатятся.

### Валидация

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Часто используемые аннотации:

| Аннотация | Что проверяет | Пример |
|----------|-------------|--------|
| `@NotNull` | Не null | ID в запросе обновления |
| `@NotBlank` | Не null, не пустая строка, не только пробелы | Имя задачи |
| `@NotEmpty` | Не null и не пустая (строка или коллекция) | Список тегов |
| `@Size(min, max)` | Длина строки или размер коллекции | `@Size(max = 255)` |
| `@Min` / `@Max` | Числовые границы | `@Min(1)` для количества |
| `@Email` | Формат email | Регистрация |
| `@Pattern(regexp)` | Регулярное выражение | Формат телефона |
| `@Positive` | Строго > 0 | Цена |

`@Valid` на параметре контроллера запускает валидацию. Если проверка не прошла — Spring бросает `MethodArgumentNotValidException` (статус 400).

### Обработка ошибок: @RestControllerAdvice

Без обработки клиент получит стандартную страницу Spring Boot (HTML «Whitelabel Error Page») или невнятный JSON. Нам нужен **единый формат ошибок**.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 404 — ресурс не найден
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(404, ex.getMessage());
    }

    // 400 — ошибка валидации
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        return new ErrorResponse(400, "Validation failed", fieldErrors);
    }

    // 409 — конфликт (например, дубликат username)
    @ExceptionHandler(DataIntegrityViolationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleConflict(DataIntegrityViolationException ex) {
        return new ErrorResponse(409, "Data conflict: resource already exists or constraint violated");
    }

    // 500 — всё остальное
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        log.error("Unexpected error", ex);
        return new ErrorResponse(500, "Internal server error");
    }
}
```

```java
public record ErrorResponse(
        int status,
        String message,
        Map<String, String> details
) {
    public ErrorResponse(int status, String message) {
        this(status, message, Map.of());
    }
}
```

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " with id " + id + " not found");
    }
}
```

Теперь все ошибки имеют единый формат:

```json
{
  "status": 400,
  "message": "Validation failed",
  "details": {
    "name": "Name is required"
  }
}
```

### HTTP-методы и статус-коды: шпаргалка

| Операция | Метод | Путь | Успех | Ошибка |
|---------|-------|------|-------|--------|
| Список | GET | /api/items?page=0&size=20 | 200 + Page | — |
| Один | GET | /api/items/42 | 200 + объект | 404 |
| Создать | POST | /api/items | 201 + объект | 400 (валидация) |
| Обновить | PUT | /api/items/42 | 200 + объект | 404 / 400 |
| Частично | PATCH | /api/items/42/status | 200 + объект | 404 / 400 |
| Удалить | DELETE | /api/items/42 | 204 (пустое тело) | 404 |

### Swagger / OpenAPI — документация бесплатно

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

Запустите приложение → http://localhost:8080/swagger-ui.html

Swagger автоматически:
- Сканирует контроллеры и генерирует документацию.
- Предоставляет UI для тестирования эндпоинтов (как Postman, но в браузере).
- Генерирует OpenAPI-спецификацию (JSON/YAML) на `/v3/api-docs`.

Улучшение документации аннотациями:

```java
@Operation(summary = "Create a new item")
@ApiResponse(responseCode = "201", description = "Item created")
@ApiResponse(responseCode = "400", description = "Invalid request")
@PostMapping
public ItemResponse create(@Valid @RequestBody CreateItemRequest request) { ... }
```

### Тестирование REST API

**Unit-тест контроллера** (без запуска сервера):

```java
@WebMvcTest(ItemController.class)
class ItemControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ItemService itemService;

    @Test
    void whenGetAll_thenReturnsPage() throws Exception {
        var page = new PageImpl<>(List.of(
            new ItemResponse(1L, "Task 1", null, "NEW", "user", LocalDateTime.now())
        ));
        when(itemService.findAll(0, 20)).thenReturn(page);

        mockMvc.perform(get("/api/items"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content[0].name").value("Task 1"))
                .andExpect(jsonPath("$.content[0].status").value("NEW"));
    }

    @Test
    void whenCreateWithBlankName_thenReturns400() throws Exception {
        mockMvc.perform(post("/api/items")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "", "description": "test"}
                    """))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.details.name").value("Name is required"));
    }

    @Test
    void whenGetNonExistent_thenReturns404() throws Exception {
        when(itemService.findById(999L))
                .thenThrow(new ResourceNotFoundException("Item", 999L));

        mockMvc.perform(get("/api/items/999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("Item with id 999 not found"));
    }
}
```

`@WebMvcTest` — поднимает только MVC-слой (контроллер + валидация + exception handler). Сервис мокается через `@MockBean`. Тест быстрый — нет ни БД, ни Tomcat.

### Задание

1. Создайте `ItemController` с полным CRUD (GET, POST, PUT, PATCH, DELETE).
2. Создайте DTO: `CreateItemRequest`, `UpdateItemRequest`, `ChangeStatusRequest`, `ItemResponse`.
3. Создайте `ItemService` с бизнес-логикой.
4. Добавьте `GlobalExceptionHandler`.
5. Добавьте валидацию в request DTO.
6. Подключите Swagger. Откройте http://localhost:8080/swagger-ui.html.
7. Напишите тесты: успешный запрос, ошибка валидации, 404.

---

## 3.5.3. Security

### Проблема: любой может удалить чужие данные

Сейчас API открыт. Любой может:
- `DELETE /api/items/1` — удалить чужую задачу.
- `POST /api/items` — создать задачу от чужого имени.
- `GET /api/items` — видеть все задачи всех пользователей.

Нужно: аутентификация (кто ты?) и авторизация (что тебе можно?).

### Два понятия, которые путают

- **Аутентификация** = «Кто ты?» → логин/пароль → получи токен.
- **Авторизация** = «Что тебе можно?» → у тебя роль USER → можешь создавать задачи, но не удалять пользователей.

### Как работает JWT-аутентификация

```
1. Клиент: POST /api/auth/login {username, password}
              │
2. Сервер:   │ проверяет пароль в БД
              │ генерирует JWT-токен (подписанный секретным ключом)
              │
3. Клиент: ◀── {token: "eyJhbGciOi..."}
              │
4. Клиент:   │ сохраняет токен (localStorage / cookie)
              │
5. Клиент: GET /api/items
           Header: Authorization: Bearer eyJhbGciOi...
              │
6. Сервер:   │ проверяет подпись токена (без обращения к БД!)
              │ извлекает username из токена
              │ проверяет права
              │
7. Клиент: ◀── [items...]
```

Ключевое: после логина **сервер не хранит сессию**. Вся информация — в токене. Это делает сервис **stateless** — можно масштабировать до 10 копий, и любая обработает запрос.

### Шаг 1: Подключение

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<!-- JWT-библиотека -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

Как только добавили `spring-boot-starter-security`, **все эндпоинты закрыты**. При попытке обратиться к API вы получите 401 или редирект на форму логина. Это правильное поведение по умолчанию — безопасность из коробки.

### Шаг 2: SecurityFilterChain

> **Важно**: `WebSecurityConfigurerAdapter` **удалён** в Spring Security 6. Если видите его в туториале — туториал устарел. Используйте `SecurityFilterChain` как бин.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/items/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

Разберём **каждую строку**, потому что тут всё важно:

- `csrf.disable()` — CSRF-защита нужна для form-based auth (браузер отправляет cookie). Для REST API с JWT-токеном в header'е CSRF бесполезен и мешает. Отключаем.
- `SessionCreationPolicy.STATELESS` — сервер **не создаёт HTTP-сессий**. Каждый запрос самодостаточен (JWT-токен содержит всю информацию). Это важно для масштабирования.
- `requestMatchers(...)` — правила авторизации. Проверяются **сверху вниз**, первое совпавшее побеждает.
- `addFilterBefore(jwtFilter, ...)` — наш фильтр проверяет JWT **до** стандартного фильтра аутентификации.

### Шаг 3: JwtService

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration-ms:86400000}")
    private long expirationMs;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }

    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
                .subject(userDetails.getUsername())
                .claim("role", userDetails.getAuthorities().iterator().next().getAuthority())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + expirationMs))
                .signWith(getSigningKey())
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername())
                && !extractClaims(token).getExpiration().before(new Date());
    }

    private Claims extractClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

### Шаг 4: JwtAuthenticationFilter

Этот фильтр выполняется на **каждый** HTTP-запрос. Его задача: если есть JWT-токен — проверить и установить аутентификацию.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");

        // Нет токена — пропускаем. Дальше Spring Security решит:
        // если эндпоинт permitAll — ок, если authenticated — 401.
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);  // убираем "Bearer "

        try {
            String username = jwtService.extractUsername(token);

            // Если username извлечён и ещё не аутентифицирован в этом запросе
            if (username != null
                    && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    var auth = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());
                    auth.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (Exception e) {
            // Невалидный токен — просто не устанавливаем аутентификацию.
            // Spring Security вернёт 401 если эндпоинт защищён.
        }

        chain.doFilter(request, response);
    }
}
```

### Шаг 5: UserDetailsService

```java
@Service
@RequiredArgsConstructor
public class AppUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        AppUser user = userRepository.findByUsername(username)
                .orElseThrow(() ->
                    new UsernameNotFoundException("User not found: " + username));
        return User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .roles(user.getRole().replace("ROLE_", ""))
                .build();
    }
}
```

### Шаг 6: AuthController

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authManager;
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public Map<String, String> register(@Valid @RequestBody RegisterRequest request) {
        if (userRepository.existsByUsername(request.username())) {
            throw new UserAlreadyExistsException(request.username());
        }
        var user = new AppUser();
        user.setUsername(request.username());
        user.setPassword(passwordEncoder.encode(request.password()));
        user.setRole("ROLE_USER");
        userRepository.save(user);
        return Map.of("message", "User registered successfully");
    }

    @PostMapping("/login")
    public Map<String, String> login(@Valid @RequestBody LoginRequest request) {
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.username(), request.password()));
        UserDetails userDetails =
            userDetailsService.loadUserByUsername(request.username());
        String token = jwtService.generateToken(userDetails);
        return Map.of("token", token);
    }
}
```

```java
public record RegisterRequest(
    @NotBlank String username,
    @NotBlank @Size(min = 6) String password
) {}

public record LoginRequest(
    @NotBlank String username,
    @NotBlank String password
) {}
```

### Шаг 7: Получение текущего пользователя

Теперь в `ItemService` мы можем знать, **кто** создаёт задачу:

```java
@Service
@RequiredArgsConstructor
public class ItemService {

    private final ItemRepository itemRepository;
    private final UserRepository userRepository;

    @Transactional
    public ItemResponse create(CreateItemRequest request,
                               String username) {  // ← передаём из контроллера
        User user = userRepository.findByUsername(username)
                .orElseThrow();
        Item item = new Item();
        item.setName(request.name());
        item.setDescription(request.description());
        item.setUser(user);
        return toResponse(itemRepository.save(item));
    }
}
```

В контроллере:

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public ItemResponse create(@Valid @RequestBody CreateItemRequest request,
                           @AuthenticationPrincipal UserDetails currentUser) {
    return itemService.create(request, currentUser.getUsername());
}
```

`@AuthenticationPrincipal` — достаёт текущего пользователя из SecurityContext. Не нужно ничего парсить вручную.

### Конфигурация

```yaml
jwt:
  secret: dGhpcyBpcyBhIHZlcnkgbG9uZyBzZWNyZXQga2V5IGZvciBqd3Q=  # base64
  expiration-ms: 86400000   # 24 часа
```

> В production `jwt.secret` берётся из переменной окружения (`JWT_SECRET`), а не из yml.

### Тестирование Security

```java
@WebMvcTest(ItemController.class)
@Import(SecurityConfig.class)
class ItemControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ItemService itemService;

    @MockBean
    private JwtService jwtService;

    @MockBean
    private UserDetailsService userDetailsService;

    @Test
    void whenNoToken_thenGetPublicEndpointReturnsOk() throws Exception {
        when(itemService.findAll(0, 20)).thenReturn(Page.empty());

        mockMvc.perform(get("/api/items"))
                .andExpect(status().isOk());
    }

    @Test
    void whenNoToken_thenPostReturns401() throws Exception {
        mockMvc.perform(post("/api/items")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "Test"}
                    """))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "testuser", roles = "USER")
    void whenAuthenticated_thenPostReturns201() throws Exception {
        var response = new ItemResponse(1L, "Test", null, "NEW", "testuser", LocalDateTime.now());
        when(itemService.create(any(), eq("testuser"))).thenReturn(response);

        mockMvc.perform(post("/api/items")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "Test"}
                    """))
                .andExpect(status().isCreated());
    }
}
```

### Задание

1. Подключите Spring Security + JWT.
2. Создайте Entity `AppUser` (id, username, password, role, createdAt).
3. Реализуйте `JwtService`, `JwtAuthenticationFilter`, `SecurityConfig`.
4. Создайте `AuthController` с `/register` и `/login`.
5. Защитите API: GET — публичный, остальные — authenticated.
6. Привяжите создание Item к текущему пользователю через `@AuthenticationPrincipal`.
7. Напишите тесты: запрос без токена (401), с токеном (201).
## 3.5.4. Deploy

### Проблема: «у меня на машине работает»

Самая частая фраза в разработке. Приложение работает локально, но при деплое:
- Другая версия Java.
- Нет PostgreSQL.
- Не те переменные окружения.
- Забыли установить зависимость.

Docker решает эту проблему: вы упаковываете приложение **со всем окружением** в образ. Образ одинаково работает на вашем ноутбуке, на сервере коллеги и в production.

### Dockerfile: правильный способ

```dockerfile
# ── Этап 1: сборка ──────────────────────────────────
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

# Сначала копируем ТОЛЬКО pom.xml и скачиваем зависимости.
# Этот слой кэшируется. Пока pom.xml не изменится —
# Docker не будет заново качать 200 МБ зависимостей.
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Теперь копируем исходники и собираем.
# Этот слой пересобирается при каждом изменении кода.
COPY src ./src
RUN mvn package -DskipTests -B

# ── Этап 2: запуск ──────────────────────────────────
FROM eclipse-temurin:21-jre
WORKDIR /app

# Копируем ТОЛЬКО jar из этапа сборки.
# Maven, исходники, .m2 — не попадают в итоговый образ.
COPY --from=build /app/target/*.jar app.jar

# Создаём непривилегированного пользователя.
# Контейнер НЕ будет работать от root.
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

Почему именно так:

**Multi-stage build** — итоговый образ содержит только JRE + JAR. Размер ~300МБ вместо ~800МБ. Нет Maven, нет исходников, нет `.m2`.

**Порядок COPY** — Docker кэширует слои. `COPY pom.xml` + `dependency:go-offline` — тяжёлый слой, но меняется редко. `COPY src` — лёгкий, но меняется при каждом коммите. Если поменять порядок — кэш не работает.

**Non-root user** — если контейнер взломан, атакующий получает права непривилегированного пользователя, а не root.

**MaxRAMPercentage** — JVM по умолчанию может попытаться взять 25% памяти контейнера под heap. С лимитом 512МБ это всего 128МБ — мало для Spring Boot. 75% — разумный баланс.

### .dockerignore

```
.git
.gitignore
.idea
*.iml
target
*.md
Dockerfile
compose.yaml
.env
```

Без `.dockerignore` команда `COPY . .` скопирует в контекст сборки папку `.git` (сотни мегабайт), `target` (десятки мегабайт), IDE-файлы. Каждый `docker build` будет медленным.

### compose.yaml

```yaml
services:
  db:
    image: postgres:17
    container_name: tracker-db
    environment:
      POSTGRES_DB: tracker
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tracker-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/tracker
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_PROFILES_ACTIVE: prod
      JWT_SECRET: ${JWT_SECRET}      # из .env файла
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

**.env** (рядом с compose.yaml, **не коммитить** в Git):

```
JWT_SECRET=dGhpcyBpcyBhIHZlcnkgbG9uZyBzZWNyZXQga2V5
```

### Graceful Shutdown

Что происходит, когда контейнер останавливается (`docker compose stop`)?

По умолчанию Docker отправляет SIGTERM, ждёт 10 секунд и убивает процесс (SIGKILL). Если в этот момент приложение обрабатывает запрос — он оборвётся.

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Теперь при SIGTERM:
1. Tomcat перестаёт принимать **новые** соединения.
2. Ждёт завершения **текущих** запросов (до 30 сек).
3. Завершает работу чисто.

### Команды

```bash
# Сборка и запуск
docker compose up -d --build

# Логи (все сервисы)
docker compose logs -f

# Логи только приложения
docker compose logs -f app

# Пересборка только приложения
docker compose up -d --build app

# Остановка
docker compose down

# Остановка с удалением данных (осторожно!)
docker compose down -v
```

### Задание

1. Создайте Dockerfile с multi-stage build.
2. Создайте `.dockerignore`.
3. Создайте `compose.yaml` (app + PostgreSQL).
4. Создайте `.env` для секретов. Добавьте `.env` в `.gitignore`.
5. Добавьте graceful shutdown.
6. `docker compose up -d --build` → приложение работает.
7. Обновите README: как запустить проект.

---

## 3.5.5. Actuator

### Проблема: приложение в production — чёрный ящик

Приложение задеплоено. Пользователи жалуются, что «медленно». Вопросы:
- Приложение вообще живо?
- База данных доступна?
- Сколько памяти ест JVM?
- Сколько запросов в секунду обрабатывается?
- Где тормозит?

Без инструментов ответ один: «не знаю, вроде работает». С **Actuator** — у вас полная картина.

### Подключение

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Health: жив ли сервис?

```bash
curl http://localhost:8080/actuator/health
```

```json
{"status": "UP"}
```

Это единственный эндпоинт, открытый по умолчанию. И самый важный: Docker, Kubernetes, балансировщики нагрузки — все проверяют здоровье сервиса через этот эндпоинт.

Включаем детали:

```yaml
management:
  endpoint:
    health:
      show-details: always    # или when-authorized для production
```

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 234567890123
      }
    }
  }
}
```

Actuator **автоматически** проверяет все компоненты, которые нашёл: БД, Redis, RabbitMQ, Kafka, дисковое пространство. Если БД недоступна — статус `DOWN`.

### Кастомный Health Indicator

Ваш сервис зависит от внешнего API? Проверяйте его:

```java
@Component
@RequiredArgsConstructor
public class PaymentServiceHealth implements HealthIndicator {

    private final RestClient paymentClient;

    @Override
    public Health health() {
        try {
            paymentClient.get().uri("/health").retrieve().toBodilessEntity();
            return Health.up()
                    .withDetail("service", "payment-api")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("service", "payment-api")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

### Другие эндпоинты

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers,env,beans,mappings
```

| Эндпоинт | Зачем | Когда полезен |
|----------|-------|-------------|
| `/actuator/health` | Жив ли сервис + компоненты | Всегда. Docker HEALTHCHECK, K8s probe |
| `/actuator/info` | Версия приложения, git commit | CI/CD — проверить, что задеплоена нужная версия |
| `/actuator/metrics` | Список метрик | Мониторинг |
| `/actuator/metrics/{name}` | Значение конкретной метрики | Отладка |
| `/actuator/loggers` | Уровни логирования | Отладка в production **без перезапуска** |
| `/actuator/env` | Конфигурация (с маскировкой секретов) | Проверить, какой профиль/конфиг загружен |
| `/actuator/beans` | Все бины в контексте | Отладка авто-конфигурации |
| `/actuator/mappings` | Все HTTP-маршруты | Проверить, что маршрут зарегистрирован |

### Метрики: Prometheus + Grafana

Actuator собирает метрики через **Micrometer** (фасад, как SLF4J для логов). Micrometer умеет экспортировать в Prometheus, Datadog, CloudWatch.

```xml
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

Теперь `/actuator/prometheus` отдаёт метрики в формате, который понимает Prometheus.

Что собирается **автоматически**:
- JVM: память (heap, non-heap), GC, потоки, классы
- HTTP: количество запросов, время ответа, статус-коды
- БД: активные соединения, время выполнения запросов
- Tomcat: потоки, сессии

### Кастомная метрика

```java
@Service
public class ItemService {

    private final Counter createCounter;
    private final Timer createTimer;

    public ItemService(MeterRegistry registry, /* ... */) {
        this.createCounter = Counter.builder("items.created.total")
                .description("Total items created")
                .register(registry);
        this.createTimer = Timer.builder("items.create.duration")
                .description("Time to create an item")
                .register(registry);
    }

    @Transactional
    public ItemResponse create(CreateItemRequest request, String username) {
        return createTimer.record(() -> {
            // ... бизнес-логика ...
            createCounter.increment();
            return toResponse(saved);
        });
    }
}
```

### Изменение логов на лету

Это **убойная фича** для production. Приложение тормозит, нужен DEBUG-лог, но перезапускать нельзя:

```bash
# Посмотреть текущий уровень
curl http://localhost:8080/actuator/loggers/ru.job4j.tracker

# Включить DEBUG без перезапуска
curl -X POST http://localhost:8080/actuator/loggers/ru.job4j.tracker \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'

# Собрали нужные логи → вернуть обратно
curl -X POST http://localhost:8080/actuator/loggers/ru.job4j.tracker \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "INFO"}'
```

### Безопасность

Actuator-эндпоинты содержат чувствительную информацию. В SecurityConfig:

```java
.requestMatchers("/actuator/health").permitAll()
.requestMatchers("/actuator/prometheus").permitAll()   // для scraper'а
.requestMatchers("/actuator/**").hasRole("ADMIN")
```

### Задание

1. Подключите Actuator. Проверьте `/actuator/health`.
2. Включите детальный health check. Остановите PostgreSQL — убедитесь, что статус стал DOWN.
3. Создайте кастомный HealthIndicator.
4. Подключите Prometheus-формат. Посмотрите `/actuator/prometheus`.
5. Добавьте кастомный Counter в ItemService.
6. Измените уровень логирования через `/actuator/loggers` **без перезапуска**.

---

## 3.5.6. Executor (асинхронность и фоновые задачи)

### Проблема: пользователь ждёт 10 секунд

Пользователь создал задачу. Сервер:
1. Сохранил в БД (50 мс).
2. Отправил email (3 секунды — внешний SMTP).
3. Отправил push-уведомление (2 секунды).
4. Обновил поисковый индекс (1 секунда).
5. Вернул ответ.

Пользователь ждал **6 секунд**. А ему нужен только пункт 1. Остальное можно делать в фоне.

### @Async: выполнение в фоновом потоке

**Шаг 1: Настроить пул потоков**

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);       // 5 потоков всегда живы
        executor.setMaxPoolSize(10);        // максимум 10
        executor.setQueueCapacity(25);      // очередь на 25 задач
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

Как работает пул:
1. Пришла задача → core-поток свободен? → выполняет.
2. Все core-потоки заняты → задача в очередь.
3. Очередь полна → создаётся новый поток (до maxPoolSize).
4. maxPoolSize достигнут, очередь полна → `CallerRunsPolicy` выполнит задачу в вызывающем потоке (вместо отказа).

**Шаг 2: Пометить метод**

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class NotificationService {

    @Async("taskExecutor")
    public void sendEmail(String to, String subject, String body) {
        log.info("[{}] Sending email to {}",
            Thread.currentThread().getName(), to);
        // ... медленная отправка ...
        log.info("[{}] Email sent to {}",
            Thread.currentThread().getName(), to);
    }
}
```

**Шаг 3: Вызвать**

```java
@PostMapping
public ItemResponse create(@Valid @RequestBody CreateItemRequest request,
                           @AuthenticationPrincipal UserDetails user) {
    ItemResponse created = itemService.create(request, user.getUsername());
    notificationService.sendEmail(           // ← возвращает СРАЗУ
        user.getUsername(),
        "Item created",
        "Your item '" + created.name() + "' has been created"
    );
    return created;   // ← пользователь получает ответ за 50мс, не за 3 секунды
}
```

### Три правила @Async, которые нарушают все

**1. Метод должен быть public.**

```java
// ❌ Не работает — Spring не может создать прокси для private
@Async
private void sendEmail(...) { }

// ✅ Работает
@Async
public void sendEmail(...) { }
```

**2. Вызов из того же класса не работает.**

```java
@Service
public class ItemService {

    @Async
    public void sendNotification(...) { }

    public void create(...) {
        // ❌ this.sendNotification() — вызов напрямую, минуя прокси.
        // Выполнится СИНХРОННО.
        this.sendNotification(...);
    }
}
```

Spring @Async работает через **прокси**. Когда вы вызываете метод через `this`, вы обходите прокси. Решение — вызывать через инъецированный бин:

```java
@Service
@RequiredArgsConstructor
public class ItemService {

    private final NotificationService notificationService;  // другой бин

    public void create(...) {
        notificationService.sendNotification(...);  // ✅ через прокси
    }
}
```

**3. Исключения теряются.**

Если async-метод бросает исключение — по умолчанию оно логируется в WARN и пропадает. Решение — глобальный обработчик:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            LoggerFactory.getLogger(method.getDeclaringClass())
                .error("Async error in {}: {}", method.getName(), ex.getMessage(), ex);
    }

    // ... getAsyncExecutor() ...
}
```

### CompletableFuture: когда нужен результат

```java
@Async("taskExecutor")
public CompletableFuture<Report> generateReport(Long userId) {
    // Долгая операция (5-10 секунд)
    Report report = buildReport(userId);
    return CompletableFuture.completedFuture(report);
}
```

```java
// Запускаем 3 отчёта параллельно
var future1 = reportService.generateReport(1L);
var future2 = reportService.generateReport(2L);
var future3 = reportService.generateReport(3L);

// Ждём все три
CompletableFuture.allOf(future1, future2, future3).join();

// Получаем результаты
Report r1 = future1.get();
Report r2 = future2.get();
Report r3 = future3.get();
```

### @Scheduled: периодические задачи

```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }
```

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class ScheduledTasks {

    private final TokenRepository tokenRepository;
    private final ReportService reportService;

    // Каждые 60 секунд: удаляем просроченные токены
    @Scheduled(fixedRate = 60_000)
    public void cleanupExpiredTokens() {
        int deleted = tokenRepository.deleteExpired();
        log.info("Cleaned up {} expired tokens", deleted);
    }

    // Каждый день в 02:00: генерируем отчёт
    @Scheduled(cron = "0 0 2 * * *")
    public void generateDailyReport() {
        log.info("Generating daily report...");
        reportService.generateDaily();
    }

    // Каждые 5 минут, но только ПОСЛЕ завершения предыдущего
    @Scheduled(fixedDelay = 300_000)
    public void syncExternalData() {
        log.info("Syncing...");
        externalService.sync();  // может занять 2 минуты
    }
}
```

`fixedRate` vs `fixedDelay`:
- `fixedRate = 60000` — запускать каждые 60 секунд от **начала** предыдущего. Если задача длится 90 секунд — следующая запустится сразу (задачи могут накладываться).
- `fixedDelay = 60000` — запускать через 60 секунд после **окончания** предыдущего. Задачи никогда не накладываются.

Cron:

```
 ┌──── секунды (0-59)
 │ ┌── минуты (0-59)
 │ │ ┌ часы (0-23)
 │ │ │ ┌ день месяца (1-31)
 │ │ │ │ ┌ месяц (1-12)
 │ │ │ │ │ ┌ день недели (0-7, MON-SUN)
 │ │ │ │ │ │
 0 0 2 * * *      каждый день в 02:00
 0 */5 * * * *    каждые 5 минут
 0 30 8 * * MON   каждый понедельник в 08:30
 0 0 0 1 * *      первое число каждого месяца в полночь
```

### Virtual Threads (Java 21)

Java 21 принесла **виртуальные потоки** — лёгкие потоки, которых можно создавать миллионами. Обычный поток платформы стоит ~1МБ памяти и привязан к потоку ОС. Виртуальный — ~1КБ и управляется JVM.

Для I/O-нагруженных приложений (REST API, обращения к БД, внешние вызовы) это огромный прирост пропускной способности.

Включение для Tomcat (одна строка!):

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Теперь каждый HTTP-запрос обрабатывается в виртуальном потоке. Вместо пула из 200 потоков Tomcat может одновременно обрабатывать тысячи запросов.

Для @Async:

```java
@Bean(name = "taskExecutor")
public Executor taskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

> Virtual Threads — не серебряная пуля. Для CPU-bound задач (расчёты, обработка изображений) обычные потоки эффективнее. Virtual Threads блестят на I/O: ожидание ответа от БД, HTTP-вызовы, чтение файлов.

### Задание

1. Настройте `ThreadPoolTaskExecutor`.
2. Создайте `NotificationService` с `@Async`-методом. Убедитесь, что контроллер отвечает мгновенно.
3. Добавьте логирование с именем потока — убедитесь, что async работает в другом потоке.
4. Создайте `@Scheduled`-задачу: удаление просроченных данных каждые 60 секунд.
5. Включите Virtual Threads. Сравните имена потоков в логах до и после.

---

## 3.5.7. Контрольные вопросы

### Введение (3.5.0)

1. Что такое Spring Boot и чем он отличается от Spring Framework?
2. Из каких трёх аннотаций состоит `@SpringBootApplication`?
3. Что такое стартер? Что произойдёт, если добавить `spring-boot-starter-data-jpa`, но не указать `spring.datasource.url`?
4. Как Spring Boot решает, какие бины создать автоматически?
5. Чем `application.yml` отличается от `application.properties`?
6. Как работают профили? Каков приоритет между `application.yml`, `application-dev.yml` и переменными окружения?
7. Почему контроллер в пакете `ru.job4j.controller` не найдётся, если `@SpringBootApplication` в `ru.job4j.tracker`?

### Data (3.5.1)

8. Что даёт `JpaRepository` из коробки? Перечислите 5 методов.
9. Как Spring Data генерирует SQL из имени метода `findByStatusAndUserOrderByCreatedAtDesc`?
10. Почему `ddl-auto: update` опасен в production? Что использовать вместо?
11. Что такое N+1 проблема? Приведите пример и решение.
12. Почему `@ManyToOne` нужно всегда ставить `FetchType.LAZY`?
13. Зачем нужен `@Enumerated(EnumType.STRING)`? Что произойдёт с `ORDINAL`, если добавить новое значение в середину enum?
14. Что такое `@PrePersist`? Приведите пример использования.
15. Как реализовать `equals`/`hashCode` для JPA Entity? Почему нельзя использовать `Objects.hash(id)`?

### REST API (3.5.2)

16. Почему нельзя возвращать Entity напрямую из контроллера? Приведите минимум 3 причины.
17. Чем `@RestController` отличается от `@Controller`?
18. Зачем нужен слой Service? Почему бизнес-логику нельзя писать в контроллере?
19. Чем Java Record подходит для DTO?
20. Как работает `@Valid`? Что произойдёт, если валидация не пройдена?
21. Что такое `@RestControllerAdvice`? Зачем нужна единая обработка ошибок?
22. Какой HTTP-статус вернуть при: создании ресурса? удалении? ошибке валидации? ресурс не найден?
23. Что тестирует `@WebMvcTest`? Что **не** тестирует?

### Security (3.5.3)

24. Чем аутентификация отличается от авторизации?
25. Нарисуйте (или опишите словами) последовательность JWT-аутентификации: от логина до запроса к защищённому эндпоинту.
26. Почему `WebSecurityConfigurerAdapter` удалён? Что используется вместо него?
27. Зачем отключать CSRF для REST API с JWT? В каких случаях CSRF нужен?
28. Что означает `SessionCreationPolicy.STATELESS`? Почему это важно для масштабирования?
29. Что делает `JwtAuthenticationFilter`? В каком месте цепочки фильтров он должен стоять?
30. Как получить текущего пользователя в контроллере?

### Deploy (3.5.4)

31. Что такое multi-stage build? Почему итоговый образ на основе JRE, а не JDK?
32. Зачем в Dockerfile сначала `COPY pom.xml` + `dependency:go-offline`, а потом `COPY src`?
33. Почему контейнер не должен работать от root?
34. Как Spring Boot подхватывает переменную окружения `SPRING_DATASOURCE_URL`?
35. Что такое graceful shutdown? Что произойдёт без него при `docker compose stop`?
36. Зачем нужен `.dockerignore`?

### Actuator (3.5.5)

37. Какие эндпоинты предоставляет Actuator? Назовите 5 и объясните зачем каждый.
38. Как Docker/Kubernetes узнаёт, что приложение живо?
39. Как создать кастомный Health Indicator?
40. Что такое Micrometer? Чем он похож на SLF4J?
41. Как изменить уровень логирования в production **без перезапуска** приложения?
42. Почему Actuator-эндпоинты нужно защищать?

### Executor (3.5.6)

43. Зачем нужна `@Async`? Какие 3 ограничения у неё есть?
44. Объясните параметры `ThreadPoolTaskExecutor`: corePoolSize, maxPoolSize, queueCapacity. Что происходит, когда все заняты?
45. В чём разница между `fixedRate` и `fixedDelay`?
46. Напишите cron-выражение для: «каждый будний день в 09:00».
47. Что такое Virtual Threads? Для каких задач они эффективны, а для каких — нет?
