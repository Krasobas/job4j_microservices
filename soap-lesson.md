# SOAP в Spring Boot: от контракта до работающего сервиса

## Что такое SOAP и зачем он нужен

SOAP (Simple Object Access Protocol) — протокол обмена сообщениями между системами. Появился в 1998 году, стал стандартом корпоративной интеграции в 2000-х. Несмотря на слово "Simple" в названии — довольно тяжеловесный.

Суть: любое сообщение — это XML-конверт с жёсткой структурой:

```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Header>
        <!-- Метаданные: авторизация, трейсинг -->
    </soap:Header>
    <soap:Body>
        <!-- Полезная нагрузка -->
        <getStudentRequest xmlns="http://job4j.ru/services/students">
            <recordBookNumber>RB-001</recordBookNumber>
        </getStudentRequest>
    </soap:Body>
</soap:Envelope>
```

Всегда XML. Всегда конверт. Всегда схема. Без исключений.

### Какие проблемы решает SOAP

**Строгий контракт.** WSDL-файл описывает: какие операции доступны, какие параметры принимают, какие типы данных используют, по какому адресу сервис живёт. Любой клиент берёт WSDL, генерирует код и вызывает сервис без документации и угадывания. В REST OpenAPI-спецификация опциональна, в SOAP — обязательна.

**Типизация.** XSD-схема описывает каждое поле: тип, обязательность, ограничения. Валидация происходит автоматически на уровне протокола. Прислал неправильный XML — получил ошибку ещё до того, как код начал выполняться.

**Транспортная независимость.** SOAP может работать поверх HTTP, JMS, SMTP, TCP. REST привязан к HTTP.

**Стандартные расширения.** WS-Security (шифрование, цифровые подписи), WS-AtomicTransaction (распределённые транзакции), WS-ReliableMessaging (гарантированная доставка).

### Почему SOAP уступил REST

Многословность (сравни `GET /students/RB-001` и XML-конверт выше), тяжёлый тулинг, WSDL нечитаем для человека, невозможно отправить запрос из браузера. В 2026 году SOAP встречается в банках, госсистемах, авиакомпаниях — везде, где legacy-интеграции живут десятилетиями. Новые сервисы на SOAP не строят, но уметь с ним работать необходимо.

### Почему в нашем проекте SOAP

Задание явно требует SOAP-интерфейс для Service S. Это проверяет умение работать с legacy-протоколами — навык, востребованный в enterprise-разработке.

---

## Главный принцип: Contract-First

В SOAP существует два подхода:

**Code-First** — пишешь Java-классы, фреймворк генерирует WSDL и XSD из них. Быстро, но контракт получается привязанным к реализации. Меняешь Java-код — ломается контракт, а значит ломаются все клиенты.

**Contract-First** — сначала описываешь контракт (XSD-схему), потом генерируешь Java-классы из него. Контракт стабилен, реализация может меняться.

Spring Web Services (Spring WS) **принципиально поддерживает только Contract-First**. Это осознанное решение авторов фреймворка: контракт — первичен, код — вторичен. В отличие от JAX-WS (стандартная Java-реализация SOAP), где можно навесить `@WebService` на класс и получить WSDL, Spring WS требует начать с XSD.

Это правильный подход. В реальных интеграциях контракт согласовывается между командами до написания кода. Если контракт зависит от реализации, любой рефакторинг становится проблемой.

---

## Шаг 0. Зависимости

Прежде чем писать любой код, убедись, что все зависимости на месте. Без любой из них цепочка сломается, и ошибки будут неочевидными.

### pom.xml

```xml
<!-- Spring Web Services — ядро: Endpoint, MessageDispatcherServlet, маршаллинг -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web-services</artifactId>
</dependency>

<!-- WSDL4J — генерация WSDL из XSD.
     БЕЗ ЭТОЙ ЗАВИСИМОСТИ DefaultWsdl11Definition падает с NoClassDefFoundError:
     javax/wsdl/extensions/ExtensibilityElement.
     Ошибка вылетает при старте приложения, не при компиляции. -->
<dependency>
    <groupId>wsdl4j</groupId>
    <artifactId>wsdl4j</artifactId>
</dependency>

<!-- JAXB API — аннотации для XML-маппинга (Jakarta namespace, обязателен с Java 11+) -->
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
</dependency>

<!-- JAXB Runtime — реализация сериализации/десериализации XML ↔ Java -->
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

Версии указывать **не нужно** — Spring Boot BOM управляет всеми четырьмя. Если явно указать несовместимую версию, можно получить конфликты.

### Maven-плагин для генерации JAXB-классов

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>jaxb2-maven-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <goals>
                        <goal>xjc</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <sources>
                    <source>src/main/resources/xsd/students.xsd</source>
                </sources>
                <packageName>ru.job4j.s.soap.gen</packageName>
                <outputDirectory>${project.build.directory}/generated-sources/jaxb</outputDirectory>
                <clearOutputDir>false</clearOutputDir>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Что зачем нужно (таблица)

| Зависимость | Зачем | Что ломается без неё |
|---|---|---|
| `spring-boot-starter-web-services` | `@Endpoint`, `@PayloadRoot`, `MessageDispatcherServlet` | Ничего из Spring WS не работает |
| `wsdl4j` | `DefaultWsdl11Definition` генерирует WSDL | `NoClassDefFoundError` при старте |
| `jakarta.xml.bind-api` | Аннотации `@XmlElement`, `@XmlRootElement` | JAXB-классы не компилируются |
| `jaxb-runtime` | Реализация маршаллинга XML ↔ Java | `ClassNotFoundException` в runtime |
| `jaxb2-maven-plugin` | Генерация Java-классов из XSD | Нет классов Request/Response |

---

## Логическая цепочка

Весь процесс — это конвейер из семи звеньев:

```
XSD → JAXB-классы → Entity+Repository → Service → Endpoint → Config → WSDL
 │         │              │                │          │          │        │
контракт  Java-код      данные           логика    обработка  сервлет  описание
данных   из контракта   из БД          маппинга   запросов   + WSDL   сервиса
```

Если убрать любое звено, цепочка рвётся. Разберём каждое.

---

## Звено 1. XSD-схема — язык контракта

XSD (XML Schema Definition) описывает структуру данных: какие элементы существуют, какого они типа, обязательны ли. Это аналог OpenAPI-спецификации для REST, но строже — валидация происходит автоматически.

### Файл `src/main/resources/xsd/students.xsd`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns:tns="http://job4j.ru/services/students"
           targetNamespace="http://job4j.ru/services/students"
           elementFormDefault="qualified">

    <!-- Запрос: получить всех студентов (без параметров) -->
    <xs:element name="getAllStudentsRequest">
        <xs:complexType/>
    </xs:element>

    <!-- Ответ: список студентов -->
    <xs:element name="getAllStudentsResponse">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="student" type="tns:studentType"
                            maxOccurs="unbounded"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <!-- Запрос: получить одного студента по номеру зачётки -->
    <xs:element name="getStudentRequest">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="recordBookNumber" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <!-- Ответ: один студент -->
    <xs:element name="getStudentResponse">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="student" type="tns:studentType"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <!-- Общий тип: данные студента -->
    <xs:complexType name="studentType">
        <xs:sequence>
            <xs:element name="recordBookNumber" type="xs:string"/>
            <xs:element name="faculty" type="xs:string"/>
            <xs:element name="lastName" type="xs:string"/>
            <xs:element name="firstName" type="xs:string"/>
            <xs:element name="middleName" type="xs:string" minOccurs="0"/>
            <xs:element name="photoKey" type="xs:string" minOccurs="0"/>
        </xs:sequence>
    </xs:complexType>

</xs:schema>
```

### Что здесь важно понять

**Namespace** (`targetNamespace="http://job4j.ru/services/students"`) — уникальный идентификатор схемы, не URL. По этому адресу ничего не лежит. Namespace нужен, чтобы различать элементы из разных схем. Если два сервиса определяют элемент `student`, namespace позволяет понять, чей именно `student` имеется в виду.

**Конвенция именования Request/Response** — Spring WS использует суффиксы `Request` и `Response` для автоматического определения операций. `getStudentRequest` + `getStudentResponse` = операция `getStudent`. Это конвенция, а не магия: Spring WS смотрит на имена элементов при генерации WSDL. Если назвать `fetchStudentQuery` и `fetchStudentResult`, автоматика не сработает и придётся настраивать маппинг вручную.

**`complexType` vs `element`** — `element` это конкретный XML-элемент, который появится в сообщении. `complexType` — это переиспользуемый тип. `studentType` описан один раз, но используется и в `getAllStudentsResponse`, и в `getStudentResponse`. Аналогия с Java: `element` — это переменная, `complexType` — это класс.

**`minOccurs="0"`** — поле необязательное. В Java это станет nullable-полем. Без этого атрибута поле обязательно (по умолчанию `minOccurs="1"`), и SOAP-запрос без него будет отклонён при валидации.

### Частые ошибки на этом этапе

Определять типы данных прямо внутри элементов вместо выделения в `complexType`. Работает, но если один и тот же тип нужен в нескольких местах — получишь дублирование.

Использовать `xs:date`, `xs:decimal` и другие сложные типы без понимания формата. Например, `xs:date` в XML это `2024-01-15`, а не `15.01.2024`. JAXB будет генерировать `XMLGregorianCalendar` вместо `LocalDate`, и с этим придётся бороться.

---

## Звено 2. JAXB — генерация Java-классов из XSD

JAXB (Jakarta XML Binding) берёт XSD-схему и создаёт Java-классы, которые умеют сериализоваться в XML и обратно. Ты не пишешь эти классы вручную — это работа плагина `jaxb2-maven-plugin`, который мы добавили в шаге 0.

После `mvn compile` в `target/generated-sources/jaxb` появятся:

```
ru/job4j/s/soap/gen/
├── GetAllStudentsRequest.java
├── GetAllStudentsResponse.java
├── GetStudentRequest.java
├── GetStudentResponse.java
├── StudentType.java
├── ObjectFactory.java
└── package-info.java
```

### Что генерируется и зачем

**`StudentType.java`** — POJO с полями из `complexType`. Каждое поле аннотировано `@XmlElement`, JAXB знает, как преобразовать его в XML и обратно.

**`GetStudentRequest.java`** — класс для входящего SOAP-сообщения. Содержит поле `recordBookNumber`. Аннотирован `@XmlRootElement`, что позволяет JAXB определить, какому XML-элементу он соответствует.

**`ObjectFactory.java`** — фабрика для создания всех сгенерированных объектов. JAXB использует её внутренне. Тебе напрямую она обычно не нужна, но Spring WS может использовать её при десериализации.

**`package-info.java`** — содержит `@XmlSchema` с namespace. Без него JAXB не знает, в каком namespace живут классы пакета.

### Важный практический момент

IDE не видит сгенерированные классы, пока не пересоберёшь проект. В IntelliJ IDEA нужно пометить `target/generated-sources/jaxb` как "Generated Sources Root" (правый клик на папке → Mark Directory As → Generated Sources Root). Иначе код будет красным, хотя компилируется нормально.

### Почему не писать классы руками?

Можно. Но тогда контракт и код рассинхронизируются. Поменял XSD — забыл обновить Java-класс — получил runtime-ошибку при десериализации. Генерация из XSD гарантирует соответствие.

---

## Звено 3. JPA-сущность и репозиторий

Прежде чем писать SOAP-логику, нужно откуда-то брать данные. В нашем проекте студенты хранятся в PostgreSQL. Создаём JPA-сущность и Spring Data репозиторий.

### Student.java

```java
package ru.job4j.s.model;

import jakarta.persistence.*;

@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "record_book_number", unique = true, nullable = false)
    private String recordBookNumber;

    @Column(nullable = false)
    private String faculty;

    @Column(name = "last_name", nullable = false)
    private String lastName;

    @Column(name = "first_name", nullable = false)
    private String firstName;

    @Column(name = "middle_name")
    private String middleName;

    @Column(name = "photo_key")
    private String photoKey;

    /* Getters and setters */

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getRecordBookNumber() { return recordBookNumber; }
    public void setRecordBookNumber(String recordBookNumber) { this.recordBookNumber = recordBookNumber; }
    public String getFaculty() { return faculty; }
    public void setFaculty(String faculty) { this.faculty = faculty; }
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getMiddleName() { return middleName; }
    public void setMiddleName(String middleName) { this.middleName = middleName; }
    public String getPhotoKey() { return photoKey; }
    public void setPhotoKey(String photoKey) { this.photoKey = photoKey; }
}
```

### StudentRepository.java

```java
package ru.job4j.s.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import ru.job4j.s.model.Student;
import java.util.Optional;

public interface StudentRepository extends JpaRepository<Student, Long> {
    Optional<Student> findByRecordBookNumber(String recordBookNumber);
}
```

---

## Звено 4. StudentService — бизнес-логика и маппинг

`StudentService` — мост между JPA-сущностью (`Student`) и JAXB-типами (`StudentType`, `GetAllStudentsResponse`, `GetStudentResponse`). Endpoint вызывает сервис, сервис достаёт данные из БД и маппит их в SOAP-ответы.

### StudentService.java

```java
package ru.job4j.s.service;

import org.springframework.stereotype.Service;
import ru.job4j.s.model.Student;
import ru.job4j.s.repository.StudentRepository;
import ru.job4j.s.soap.gen.*;
import java.util.List;

@Service
public class StudentService {

    private final StudentRepository repository;

    public StudentService(StudentRepository repository) {
        this.repository = repository;
    }

    /* ===== Методы для работы с JPA-сущностями ===== */

    public List<Student> findAll() {
        return repository.findAll();
    }

    public Student findByRecordBookNumber(String number) {
        return repository.findByRecordBookNumber(number)
                .orElseThrow(() -> new StudentNotFoundException(number));
    }

    /* ===== Методы для SOAP: возвращают JAXB-типы ===== */

    public GetAllStudentsResponse getAllStudentsAsXml() {
        var response = new GetAllStudentsResponse();
        findAll().stream()
                .map(this::toStudentType)
                .forEach(response.getStudent()::add);
        return response;
    }

    public GetStudentResponse getStudentAsXml(String recordBookNumber) {
        var response = new GetStudentResponse();
        response.setStudent(toStudentType(findByRecordBookNumber(recordBookNumber)));
        return response;
    }

    /* ===== Маппинг: JPA-сущность → JAXB-тип ===== */

    private StudentType toStudentType(Student student) {
        var type = new StudentType();
        type.setRecordBookNumber(student.getRecordBookNumber());
        type.setFaculty(student.getFaculty());
        type.setLastName(student.getLastName());
        type.setFirstName(student.getFirstName());
        type.setMiddleName(student.getMiddleName());
        type.setPhotoKey(student.getPhotoKey());
        return type;
    }
}
```

### Почему маппинг в сервисе, а не в Endpoint?

Endpoint — тонкий адаптер протокола. Он не знает про JPA-сущности. Сервис — единственное место, где пересекаются два мира: JPA и JAXB. Если завтра добавится REST-контроллер, он тоже будет вызывать `findAll()`, но не `getAllStudentsAsXml()`.

### StudentNotFoundException.java

```java
package ru.job4j.s.service;

public class StudentNotFoundException extends RuntimeException {
    public StudentNotFoundException(String recordBookNumber) {
        super("Student not found: " + recordBookNumber);
    }
}
```

---

## Звено 5. Endpoint — обработка SOAP-запросов

Endpoint — аналог `@RestController` в REST. Метод принимает десериализованный запрос и возвращает ответ, который Spring WS сериализует обратно в XML и оборачивает в SOAP-конверт.

### StudentEndpoint.java

```java
package ru.job4j.s.soap;

import org.springframework.ws.server.endpoint.annotation.Endpoint;
import org.springframework.ws.server.endpoint.annotation.PayloadRoot;
import org.springframework.ws.server.endpoint.annotation.RequestPayload;
import org.springframework.ws.server.endpoint.annotation.ResponsePayload;
import ru.job4j.s.service.StudentService;
import ru.job4j.s.soap.gen.*;

@Endpoint
public class StudentEndpoint {

    private static final String NAMESPACE = "http://job4j.ru/services/students";

    private final StudentService studentService;

    public StudentEndpoint(StudentService studentService) {
        this.studentService = studentService;
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "getAllStudentsRequest")
    @ResponsePayload
    public GetAllStudentsResponse getAll(
            @RequestPayload GetAllStudentsRequest request) {
        return studentService.getAllStudentsAsXml();
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "getStudentRequest")
    @ResponsePayload
    public GetStudentResponse getOne(
            @RequestPayload GetStudentRequest request) {
        return studentService.getStudentAsXml(request.getRecordBookNumber());
    }
}
```

### Маппинг запроса на метод

В REST маппинг идёт по HTTP-методу и URL: `@GetMapping("/students/{id}")`.

В SOAP маппинг идёт по **содержимому XML**: namespace + имя корневого элемента.

`@PayloadRoot(namespace = "...", localPart = "getStudentRequest")` — означает: "если пришёл SOAP-запрос, внутри которого XML-элемент `getStudentRequest` из namespace `http://job4j.ru/services/students`, вызови этот метод".

Spring WS парсит XML-тело запроса, находит корневой элемент, сопоставляет его с `@PayloadRoot` и вызывает нужный метод.

**`@RequestPayload`** — десериализует XML-тело в Java-объект (через JAXB).

**`@ResponsePayload`** — сериализует возвращённый объект обратно в XML.

### Разделение ответственности

Endpoint **не содержит бизнес-логики**. Он только принимает запрос, вызывает сервис и возвращает результат. Вся логика — в `StudentService`. Endpoint — тонкий адаптер между SOAP-протоколом и бизнес-логикой. Точно так же, как REST-контроллер не должен содержать SQL-запросы.

---

## Звено 6. WebServiceConfig — сборка инфраструктуры

Конфигурационный класс регистрирует сервлет для SOAP-запросов и настраивает генерацию WSDL.

### WebServiceConfig.java

```java
package ru.job4j.s.config.soap;

import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.ws.config.annotation.EnableWs;
import org.springframework.ws.transport.http.MessageDispatcherServlet;
import org.springframework.ws.wsdl.wsdl11.DefaultWsdl11Definition;
import org.springframework.xml.xsd.SimpleXsdSchema;
import org.springframework.xml.xsd.XsdSchema;

@EnableWs
@Configuration
public class WebServiceConfig {

    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(
            ApplicationContext ctx) {
        var servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(ctx);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }

    @Bean(name = "students")
    public DefaultWsdl11Definition studentsWsdl(XsdSchema studentsSchema) {
        var definition = new DefaultWsdl11Definition();
        definition.setPortTypeName("StudentsPort");
        definition.setLocationUri("/ws");
        definition.setTargetNamespace("http://job4j.ru/services/students");
        definition.setSchema(studentsSchema);
        return definition;
    }

    @Bean
    public XsdSchema studentsSchema() {
        return new SimpleXsdSchema(new ClassPathResource("xsd/students.xsd"));
    }
}
```

### Разбор по строкам

**`@EnableWs`** — включает инфраструктуру Spring Web Services. Без этой аннотации `@Endpoint` и `@PayloadRoot` не будут работать.

**`MessageDispatcherServlet`** — аналог `DispatcherServlet` из Spring MVC, но для SOAP. Он принимает HTTP-запрос, извлекает XML из SOAP-конверта, находит нужный `@Endpoint` и вызывает его. Регистрируется на путь `/ws/*`, то есть все SOAP-запросы идут на `http://host:port/ws/`.

**`setTransformWsdlLocations(true)`** — критически важная настройка. WSDL содержит адрес сервиса. Без этого флага в WSDL будет жёстко прописан `http://localhost:8081/ws`. С этим флагом Spring WS автоматически подставляет адрес из входящего запроса. Если запрос пришёл на `http://my-server.com/ws`, в WSDL будет `http://my-server.com/ws`. Без этого клиенты не смогут подключиться к сервису, развёрнутому не на localhost.

**`@Bean(name = "students")`** — имя бина определяет URL, по которому доступен WSDL. При имени `"students"` WSDL лежит по адресу `http://host:port/ws/students.wsdl`. Если назвать бин `"studentService"`, WSDL будет на `/ws/studentService.wsdl`.

**`setPortTypeName("StudentsPort")`** — имя порта в WSDL. Клиент использует его для идентификации. Не влияет на работу, но должно быть осмысленным.

**`setLocationUri("/ws")`** — базовый путь для SOAP-эндпоинтов. Должен совпадать с путём `MessageDispatcherServlet`.

**`setTargetNamespace(...)`** — namespace в WSDL. Должен совпадать с `targetNamespace` в XSD.

### Что Spring WS делает с XSD

Он анализирует имена элементов и по конвенции суффиксов `Request`/`Response` определяет операции:

```
getAllStudentsRequest  + getAllStudentsResponse  → операция getAllStudents
getStudentRequest     + getStudentResponse      → операция getStudent
```

Результат — готовый WSDL с двумя операциями, доступный по URL.

### Практическая проверка

После запуска сервиса открой `http://localhost:8081/ws/students.wsdl` в браузере. Ты увидишь сгенерированный WSDL с описанием операций, типов и адреса сервиса. Если WSDL не генерируется, обычно причина в одном из трёх: namespace в `DefaultWsdl11Definition` не совпадает с namespace в XSD, имена элементов не следуют конвенции `Request`/`Response`, или XSD содержит синтаксическую ошибку.

---

## Звено 7. Маршаллинг — сборка XML из Java и обратно

JAXB-классы, сгенерированные из XSD, аннотированы так, что JAXB умеет конвертировать их в XML и обратно. Spring WS делает это автоматически для Endpoint.

Но в нашем проекте маршаллинг нужен ещё в одном месте: в Kafka-слушателе. Когда Service S получает запрос из Kafka, он должен вернуть XML-строку, а не Java-объект. Kafka не знает про JAXB.

### JaxbMarshaller.java

```java
package ru.job4j.s.soap;

import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.JAXBException;
import jakarta.xml.bind.Marshaller;
import org.springframework.stereotype.Component;
import ru.job4j.s.soap.gen.GetAllStudentsResponse;
import ru.job4j.s.soap.gen.GetStudentResponse;
import java.io.StringWriter;

@Component
public class JaxbMarshaller {

    private final JAXBContext context;

    public JaxbMarshaller() throws JAXBException {
        this.context = JAXBContext.newInstance(
                GetAllStudentsResponse.class,
                GetStudentResponse.class
        );
    }

    public String marshal(Object object) {
        try {
            Marshaller marshaller = context.createMarshaller();
            marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
            marshaller.setProperty(Marshaller.JAXB_FRAGMENT, true);
            StringWriter writer = new StringWriter();
            marshaller.marshal(object, writer);
            return writer.toString();
        } catch (JAXBException e) {
            throw new RuntimeException("Failed to marshal object", e);
        }
    }
}
```

**`JAXB_FORMATTED_OUTPUT`** — делает XML читаемым (с отступами). В продакшене можно отключить для экономии трафика.

**`JAXB_FRAGMENT`** — не добавляет XML-декларацию (`<?xml version="1.0"?>`). Для передачи через Kafka это удобнее, потому что получатель может сам решить, нужна ли она.

**`JAXBContext` создаётся один раз в конструкторе** — это тяжёлая операция. Создавать его на каждый запрос — расточительство. `JAXBContext` потокобезопасен. `Marshaller` — нет, поэтому создаём новый `Marshaller` на каждый вызов.

---

## Итоговая структура файлов

```
service-s/src/main/
├── java/ru/job4j/s/
│   ├── ServiceSApplication.java
│   ├── config/soap/
│   │   └── WebServiceConfig.java         ← Звено 6
│   ├── model/
│   │   └── Student.java                  ← Звено 3
│   ├── repository/
│   │   └── StudentRepository.java        ← Звено 3
│   ├── service/
│   │   ├── StudentService.java           ← Звено 4
│   │   └── StudentNotFoundException.java ← Звено 4
│   └── soap/
│       ├── StudentEndpoint.java          ← Звено 5
│       ├── JaxbMarshaller.java           ← Звено 7
│       └── gen/                          ← Звено 2 (генерируется)
│           ├── GetAllStudentsRequest.java
│           ├── GetAllStudentsResponse.java
│           ├── GetStudentRequest.java
│           ├── GetStudentResponse.java
│           ├── StudentType.java
│           └── ObjectFactory.java
└── resources/
    └── xsd/
        └── students.xsd                  ← Звено 1
```

---

## Обработка ошибок

### SOAP Fault

В REST ошибка — это HTTP-статус (404, 500) и JSON с описанием. В SOAP ошибка — специальный XML-элемент `Fault` внутри конверта:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <soap:Fault>
            <faultcode>soap:Server</faultcode>
            <faultstring>Student not found: RB-999</faultstring>
        </soap:Fault>
    </soap:Body>
</soap:Envelope>
```

### Реализация через Exception Resolver

Бросить исключение из Endpoint — Spring WS обернёт его в generic SOAP Fault. Работает, но клиент получит невнятное сообщение. Для читаемых ошибок — кастомный resolver:

```java
package ru.job4j.s.config.soap;

import org.springframework.ws.soap.server.endpoint.SoapFaultDefinition;
import org.springframework.ws.soap.server.endpoint.SoapFaultMappingExceptionResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import ru.job4j.s.service.StudentNotFoundException;
import java.util.Properties;

@Configuration
public class SoapFaultConfig {

    @Bean
    public SoapFaultMappingExceptionResolver exceptionResolver() {
        var resolver = new SoapFaultMappingExceptionResolver();
        var defaultFault = new SoapFaultDefinition();
        defaultFault.setFaultCode(SoapFaultDefinition.SERVER);
        resolver.setDefaultFault(defaultFault);
        var props = new Properties();
        props.setProperty(
                StudentNotFoundException.class.getName(),
                SoapFaultDefinition.SERVER.toString()
        );
        resolver.setExceptionMappings(props);
        resolver.setOrder(1);
        return resolver;
    }
}
```

---

## Тестирование SOAP-сервиса

### Из командной строки через curl

SOAP-запрос — обычный HTTP POST с XML-телом:

```bash
curl -X POST http://localhost:8081/ws \
  -H "Content-Type: text/xml" \
  -d '
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:stu="http://job4j.ru/services/students">
    <soapenv:Body>
        <stu:getAllStudentsRequest/>
    </soapenv:Body>
</soapenv:Envelope>'
```

Для конкретного студента:

```bash
curl -X POST http://localhost:8081/ws \
  -H "Content-Type: text/xml" \
  -d '
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:stu="http://job4j.ru/services/students">
    <soapenv:Body>
        <stu:getStudentRequest>
            <stu:recordBookNumber>RB-001</stu:recordBookNumber>
        </stu:getStudentRequest>
    </soapenv:Body>
</soapenv:Envelope>'
```

### Интеграционный тест на Spring WS

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class StudentEndpointTest {

    @Autowired
    private ApplicationContext ctx;

    private MockWebServiceClient client;

    @BeforeEach
    void setUp() {
        client = MockWebServiceClient.createClient(ctx);
    }

    @Test
    void getAllStudentsShouldReturnList() {
        Source request = new StringSource(
                "<getAllStudentsRequest xmlns='http://job4j.ru/services/students'/>"
        );
        client.sendRequest(RequestCreators.withPayload(request))
                .andExpect(ResponseMatchers.noFault())
                .andExpect(ResponseMatchers.xpath(
                        "//ns:student",
                        Map.of("ns", "http://job4j.ru/services/students")
                ).exists());
    }
}
```

`MockWebServiceClient` — Spring WS аналог `MockMvc` для REST. Отправляет SOAP-запрос напрямую в `MessageDispatcherServlet`, минуя сеть.

---

## Распространённые ошибки

### 1. Забыли wsdl4j

Без `wsdl4j` в зависимостях приложение **падает при старте** с ошибкой:

```
NoClassDefFoundError: javax/wsdl/extensions/ExtensibilityElement
```

Ошибка вылетает из `DefaultWsdl11Definition.<init>()`. При этом компиляция проходит успешно — класс нужен только в runtime. Это самая коварная проблема: код компилируется, IDE не подсвечивает ошибок, но приложение не стартует.

### 2. Namespace mismatch

Namespace в XSD, в `@PayloadRoot`, в `DefaultWsdl11Definition` и в SOAP-запросе клиента должны совпадать **символ в символ**. Trailing slash имеет значение: `http://job4j.ru/services/students` и `http://job4j.ru/services/students/` — это разные namespace. Самая частая причина "SOAP endpoint not found".

### 3. Имена элементов не следуют конвенции

Если назвать элемент `getStudentQuery` вместо `getStudentRequest`, Spring WS не сможет автоматически создать операцию `getStudent` в WSDL. `DefaultWsdl11Definition` ожидает суффиксы `Request` и `Response`. Можно настроить другие суффиксы:

```java
definition.setRequestSuffix("Query");
definition.setResponseSuffix("Result");
```

Но лучше следовать конвенции — меньше неожиданностей.

### 4. `@Endpoint` не сканируется

`@Endpoint` — это `@Component`. Если класс вне пакета, который сканирует Spring Boot, он не подхватится. Убедись, что пакет Endpoint-а является подпакетом главного класса приложения.

### 5. JAXB-классы не в classpath

Плагин `jaxb2-maven-plugin` генерирует классы в `target/generated-sources/`. Maven автоматически добавляет эту директорию в source path, но Gradle — нет (нужна дополнительная конфигурация). При сборке через IDE без Maven тоже могут быть проблемы — классы не видны, проект не компилируется.

### 6. Lombok @RequiredArgsConstructor с @Value

Если используешь Lombok `@RequiredArgsConstructor` и `@Value("${...}")` на `final`-поле, Lombok сгенерирует конструктор **без** `@Value` на параметре. Spring не сможет инжектить значение из properties и попросит бин типа `String`. Решение — писать конструктор вручную для классов с `@Value`-параметрами.

---

## Полная картина: что происходит при SOAP-запросе

```
1. Клиент отправляет HTTP POST на /ws с XML-телом

2. MessageDispatcherServlet принимает запрос
   ↓
3. SoapMessageFactory парсит HTTP-тело в SOAP-конверт
   ↓
4. PayloadRootAnnotationMethodEndpointMapping
   извлекает namespace + localPart из Body
   → находит метод с подходящим @PayloadRoot
   ↓
5. JAXB Unmarshaller конвертирует XML → Java-объект
   (@RequestPayload)
   ↓
6. Вызывается метод Endpoint-а
   → Endpoint зовёт StudentService
   → StudentService зовёт StudentRepository
   → Repository достаёт данные из PostgreSQL
   → StudentService маппит Student → StudentType
   → Формируется GetAllStudentsResponse
   ↓
7. JAXB Marshaller конвертирует Java-объект → XML
   (@ResponsePayload)
   ↓
8. Spring WS оборачивает XML в SOAP-конверт
   ↓
9. HTTP-ответ с XML-телом уходит клиенту
```

Каждый шаг отделён от остальных. XSD определяет контракт, JAXB обеспечивает сериализацию, Spring WS управляет маршрутизацией, Endpoint вызывает Service, Service работает с Repository. Если нужно поменять хранилище с PostgreSQL на MongoDB, меняется только Repository и Service — XSD, Endpoint, WSDL остаются прежними. Контракт стабилен.
