# Entendiendo la Arquitectura de la Aplicación

## Preguntas Clave

1. **¿Cuáles son los componentes funcionales de la aplicación y cómo se relacionan entre sí?**
2. **¿Cómo es el despliegue de los componentes en el entorno productivo?**
3. **¿Cómo interactúan los componentes con las fuentes de datos?**
4. **¿Qué patrones y tácticas de arquitectura se están utilizando?**
5. **¿Qué tecnologías y frameworks forman parte de la arquitectura?**
6. **¿Cuáles son los principales módulos o capas en la aplicación?**
7. **¿Existen dependencias entre los servicios o microservicios?**
8. **¿Cómo se gestionan la seguridad y la autenticación dentro de la aplicación?**
9. **¿Existen mecanismos de escalabilidad y balanceo de carga?**
10. **¿Cómo se manejan los errores y la resiliencia del sistema?**
11. **¿Cómo se entienden las capas de la aplicación y cómo se manejan?**
12. **¿Cómo me puedo comunicar con esta aplicación?: API? mecanismo de comunicación. Si es un api generar el código para entender cuales son los endpoints**

---

## Respuestas

### 1. **¿Cuáles son los componentes funcionales de la aplicación y cómo se relacionan entre sí?**

La aplicación **Bookstore** es un sistema de gestión de una tienda de libros que opera con una arquitectura monolítica basada en componentes funcionales claramente definidos:

#### **Componentes Principales:**

- **Gestión de Libros (Books)**: Componente central que administra el catálogo de libros, incluyendo información como ISBN, descripción, imagen y fecha de publicación.

- **Gestión de Autores (Authors)**: Gestiona información de autores, incluyendo biografía, imagen y fecha de nacimiento.

- **Gestión de Editoriales (Editorials)**: Administra las editoriales responsables de publicar los libros.

- **Gestión de Premios (Prizes)**: Sistema de premios y reconocimientos otorgados a autores por organizaciones.

- **Gestión de Organizaciones (Organizations)**: Entidades que otorgan premios a autores.

- **Gestión de Reseñas (Reviews)**: Sistema de comentarios y reseñas de libros de diferentes fuentes.

#### **Relaciones entre Componentes:**

```
┌─────────────────────────────────────────────────────────┐
│                    BOOKSTORE APPLICATION                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │              EDITORIAL (1)                       │    │
│  │         Publica múltiples libros                 │    │
│  └────────────────────┬─────────────────────────────┘    │
│                       │ (1:N)                             │
│                       ▼                                   │
│  ┌──────────────────────────────────────────────────┐    │
│  │              BOOK (Núcleo)                       │    │
│  │    ISBN, Descripción, Imagen, Fecha Pub.         │    │
│  └────────┬─────────────────────────────┬───────────┘    │
│           │ (N:M)                       │ (1:N)           │
│           ▼                             ▼                │
│  ┌──────────────────────┐     ┌──────────────────────┐   │
│  │   AUTHOR (N:M)       │     │   REVIEW             │   │
│  │ Nombre, Biografía    │     │ Descripción, Fuente  │   │
│  └──────────┬───────────┘     └──────────────────────┘   │
│             │ (1:N)                                       │
│             ▼                                             │
│  ┌──────────────────────┐                                │
│  │   PRIZE              │                                │
│  │ Nombre, Descripción  │                                │
│  └──────────┬───────────┘                                │
│             │ (1:1)                                       │
│             ▼                                             │
│  ┌──────────────────────┐                                │
│  │  ORGANIZATION        │                                │
│  │ Nombre, Tipo         │                                │
│  └──────────────────────┘                                │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

#### **Dependencias entre Componentes:**

- **Libro → Editorial**: Dependencia obligatoria (ManyToOne). Cada libro debe estar asociado a una editorial.
- **Libro ↔ Autor**: Relación muchos-a-muchos bidireccional. Un libro puede tener múltiples autores y un autor puede haber escrito múltiples libros.
- **Libro → Review**: Dependencia de contenido. Las reseñas son dependientes del libro.
- **Autor → Prize**: Relación uno-a-muchos. Un autor puede recibir múltiples premios.
- **Prize → Organization**: Relación uno-a-uno. Cada premio está asociado a una organización que lo otorga.

---

### 2. **¿Cómo es el despliegue de los componentes en el entorno productivo?**

#### **Estrategia de Despliegue:**

La aplicación **está preparada para despliegue en contenedores Docker**, aunque no hay evidencia de orquestación en producción (como Kubernetes o Docker Swarm).

#### **Proceso de Despliegue - Dockerfile:**

La aplicación incluye un `Dockerfile` con un enfoque **multi-stage build**:

```dockerfile
# DOCKERFILE - Multi-stage Build
FROM maven:3.9.8-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jdk-alpine
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

**Características del Dockerfile:**

1. **Multi-stage Build**: Reduce el tamaño de la imagen final eliminando las herramientas de construcción (Maven).
2. **JDK Alpine Linux**: Utiliza una imagen base ligera basada en Alpine Linux para minimizar el footprint.
3. **Java 21**: Runtime de Java 21 con soporte de características modernas.
4. **JAR ejecutable**: Spring Boot produce un JAR autoejecutables sin necesidad de servidor de aplicaciones externo.

#### **Pipeline CI/CD - Jenkins:**

La aplicación utiliza **Jenkins** como herramienta de integración continua. El pipeline utiliza **contenedores Docker para las etapas de construcción y testing**, pero esto es para el proceso de CI/CD, no necesariamente para producción:

```
Git Checkout → Build (Docker) → Testing (Docker) → SonarQube Analysis → ARCC Analysis
```

**Stages principales:**

1. **Checkout**: Descarga el código del repositorio GitHub
2. **GitInspector**: Análisis de calidad del código
3. **Build**: Compilación con Maven dentro de contenedor Docker (`citools-isis2603:latest`)
4. **Testing**: Ejecución de pruebas unitarias e integración dentro de contenedor
5. **Static Analysis**: Análisis estático con SonarQube
6. **ARCC**: Análisis de componentes arquitectónicos

#### **Infraestructura y Herramientas:**

- **Dockerfile**: Prepara la aplicación para despliegue en contenedores
- **Jenkins**: Orquesta el pipeline CI/CD (ubicado en ambiente de desarrollo)
- **SonarQube**: Análisis de calidad de código en `http://172.24.101.209:8082/sonar-isis2603`
- **Maven**: Gestor de dependencias y build
- **H2 Database**: Base de datos en memoria para desarrollo/testing (debe ser reemplazada en producción por PostgreSQL, MySQL, etc.)

#### **Nota Importante:**

No hay evidencia de archivos como `docker-compose.yml`, `Kubernetes manifests`, o configuración de orquestación. El `Dockerfile` permite crear una imagen, pero el despliegue final depende de la infraestructura disponible.

---

### 3. **¿Cómo interactúan los componentes con las fuentes de datos?**

#### **Capas de Acceso a Datos:**

La aplicación implementa el patrón **Repository** para la abstracción del acceso a datos:

```
Controlador → Servicio → Repository → ORM (Hibernate) → Database
```

#### **Configuración de Base de Datos:**

```properties
# application.properties
spring.datasource.url=jdbc:h2:mem:bookstore
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.show-sql=false
spring.jpa.hibernate.ddl-auto=none
server.servlet.context-path=/api
spring.jpa.open-in-view=true
spring.sql.init.schema-locations=classpath:sql/schema.sql
spring.sql.init.data-locations=classpath:sql/data.sql
```

#### **Tecnología ORM:**

- **Spring Data JPA**: Proporciona abstracción sobre Hibernate
- **Hibernate**: ORM que mapea entidades Java a tablas de base de datos
- **H2 Database**: Base de datos en memoria para desarrollo (puede ser reemplazada por PostgreSQL, MySQL, etc. en producción)

#### **Repositorios:**

Cada entidad tiene su correspondiente repositorio que extiende de `JpaRepository`:

```java
public interface BookRepository extends JpaRepository<BookEntity, Long> {
    List<BookEntity> findByIsbn(String isbn);
}

public interface AuthorRepository extends JpaRepository<AuthorEntity, Long> {
}

public interface EditorialRepository extends JpaRepository<EditorialEntity, Long> {
}

public interface PrizeRepository extends JpaRepository<PrizeEntity, Long> {
}

public interface OrganizationRepository extends JpaRepository<OrganizationEntity, Long> {
}

public interface ReviewRepository extends JpaRepository<ReviewEntity, Long> {
}
```

#### **Transacciones:**

Los servicios utilizan anotación `@Transactional` para gestionar el ciclo de vida de las transacciones:

```java
@Transactional
public BookEntity createBook(BookEntity bookEntity) {
    // Validaciones y creación
    return bookRepository.save(bookEntity);
}
```

#### **Inicialización de Datos:**

- **schema.sql**: Define la estructura de las tablas
- **data.sql**: Carga datos iniciales en la base de datos
- **Lazy Loading**: Utiliza `FetchType.LAZY` para optimizar consultas

---

### 4. **¿Qué patrones y tácticas de arquitectura se están utilizando?**

#### **Patrones Arquitectónicos Principales:**

1. **Arquitectura en Capas (Layered Architecture)**
   - Separación clara entre presentación, lógica de negocio y datos
   - Cada capa tiene responsabilidades bien definidas

2. **Patrón Repository**
   - Abstracción del acceso a datos
   - Facilita testing y cambios en la fuente de datos
   - Repositorios específicos para cada entidad

3. **Patrón Service/Business Logic**
   - Centralización de la lógica de negocio
   - Servicios inyectados en controladores
   - Validaciones y reglas de negocio en servicios

4. **Data Transfer Object (DTO)**
   - Separación entre entidades persistentes y DTOs
   - Previene exposición de detalles internos
   - Permite transformación de datos (`ModelMapper`)

5. **Inyección de Dependencias (Dependency Injection)**
   - Uso extensivo de `@Autowired`
   - Componentes desacoplados y testeables

6. **Exception Handling Centralizado**
   - `@ControllerAdvice` para manejo global de excepciones
   - Respuestas consistentes para errores

#### **Tácticas de Seguridad:**

1. **CORS (Cross-Origin Resource Sharing)**
   - Configuración permisiva para desarrollo
   - Métodos permitidos: GET, POST, PUT, DELETE

2. **Validación de Datos**
   - Validaciones en servicios (ISBN, Editorial)
   - Prevención de violaciones de restricciones

#### **Tácticas de Rendimiento:**

1. **Lazy Loading**
   - Carga de datos bajo demanda
   - Reducciona de uso de memoria

2. **Session Management**
   - `spring.jpa.open-in-view=true`: Permite lazy loading fuera de la transacción

#### **Tácticas de Testing:**

1. **Separación de pruebas**
   - Unit tests: Pruebas de servicios
   - Integration tests: Pruebas end-to-end
   - Coverage measurement: JaCoCo

2. **Data Generation**
   - PODAM para generación de datos de prueba
   - Custom strategies para tipos específicos (dates)

---

### 5. **¿Qué tecnologías y frameworks forman parte de la arquitectura?**

#### **Stack Tecnológico:**

| Componente | Tecnología | Versión |
|-----------|-----------|---------|
| **Framework Web** | Spring Boot | 3.3.3 |
| **Lenguaje** | Java | 21 |
| **ORM** | JPA/Hibernate | (incluido en Spring) |
| **Acceso a Datos** | Spring Data JPA | (incluido en Spring) |
| **Database** | H2 (desarrollo) | - |
| **Contenedorización** | Docker | 3.9.8 (Maven), Eclipse Temurin |
| **Mapeo de Objetos** | ModelMapper | 2.3.5 |
| **Annotations** | Lombok | - |
| **Generación de Datos** | PODAM | 7.2.7.RELEASE |
| **Testing** | JUnit/Spring Boot Test | (incluido) |
| **Coverage** | JaCoCo | 0.8.11 |
| **Build Tool** | Maven | 3.9.8 |
| **CI/CD** | Jenkins | - |
| **Quality Analysis** | SonarQube | - |
| **Build Image Base** | Eclipse Temurin | 21-jdk-alpine |

#### **Dependencias Principales:**

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Jakarta Persistence API -->
    <dependency>
        <groupId>jakarta.persistence</groupId>
        <artifactId>jakarta.persistence-api</artifactId>
    </dependency>
    
    <!-- H2 Database -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
    </dependency>
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
    
    <!-- ModelMapper -->
    <dependency>
        <groupId>org.modelmapper</groupId>
        <artifactId>modelmapper</artifactId>
        <version>2.3.5</version>
    </dependency>
    
    <!-- PODAM for Test Data Generation -->
    <dependency>
        <groupId>uk.co.jemos.podam</groupId>
        <artifactId>podam</artifactId>
        <version>7.2.7.RELEASE</version>
    </dependency>
</dependencies>
```

---

### 6. **¿Cuáles son los principales módulos o capas en la aplicación?**

#### **Estructura de Capas:**

```
BookstoreApplication (Punto de entrada)
│
├── Controllers (Capa de Presentación)
│   ├── BookController
│   ├── AuthorController
│   ├── EditorialController
│   ├── ReviewController
│   ├── PrizeController
│   ├── OrganizationController
│   ├── BookAuthorController
│   ├── AuthorBookController
│   ├── BookEditorialController
│   ├── EditorialBookController
│   ├── PrizeAuthorController
│   └── DefaultController
│
├── Services (Capa de Lógica de Negocio)
│   ├── BookService
│   ├── AuthorService
│   ├── EditorialService
│   ├── ReviewService
│   ├── PrizeService
│   ├── OrganizationService
│   ├── BookAuthorService
│   ├── AuthorBookService
│   ├── BookEditorialService
│   ├── EditorialBookService
│   └── PrizeAuthorService
│
├── Repositories (Capa de Acceso a Datos)
│   ├── BookRepository
│   ├── AuthorRepository
│   ├── EditorialRepository
│   ├── ReviewRepository
│   ├── PrizeRepository
│   └── OrganizationRepository
│
├── Entities (Modelo de Dominio)
│   ├── BaseEntity (Clase abstracta)
│   ├── BookEntity
│   ├── AuthorEntity
│   ├── EditorialEntity
│   ├── ReviewEntity
│   ├── PrizeEntity
│   └── OrganizationEntity
│
├── DTOs (Transferencia de Datos)
│   ├── BookDTO & BookDetailDTO
│   ├── AuthorDTO & AuthorDetailDTO
│   ├── EditorialDTO & EditorialDetailDTO
│   ├── ReviewDTO
│   ├── PrizeDTO & PrizeDetailDTO
│   ├── OrganizationDTO & OrganizationDetailDTO
│
├── Exceptions (Manejo de Errores)
│   ├── EntityNotFoundException
│   ├── IllegalOperationException
│   ├── ApiError
│   └── RestExceptionHandler
│
├── Config (Configuración)
│   └── ApplicationConfig
│       ├── ModelMapper Bean
│       └── CORS Configuration
│
└── Utilities
    └── PODAM
        ├── DateStrategy
        └── (otras estrategias de generación)
```

#### **Descripción de Capas:**

1. **Capa de Presentación (Controllers)**
   - Maneja solicitudes HTTP
   - Valida entrada de usuario
   - Convierte DTOs a entidades y viceversa
   - Retorna respuestas HTTP

2. **Capa de Lógica de Negocio (Services)**
   - Implementa reglas de negocio
   - Validaciones complejas
   - Transacciones
   - Orquestación de operaciones

3. **Capa de Acceso a Datos (Repositories)**
   - Consultas a base de datos
   - Abstracción de ORM
   - Métodos personalizados de consulta

4. **Capa de Modelo de Dominio (Entities)**
   - Representación de entidades de negocio
   - Mapeo a tablas de base de datos
   - Relaciones entre entidades

5. **Capa de Transferencia de Datos (DTOs)**
   - Separación de preocupaciones
   - Exposición segura de datos
   - Transformación de datos

---

### 7. **¿Existen dependencias entre los servicios o microservicios?**

#### **Naturaleza de la Aplicación:**

**No es una arquitectura de microservicios.** Es una aplicación monolítica con un único servicio cohesivo.

#### **Dependencias Internas:**

Las dependencias existen entre servicios dentro del mismo proceso:

1. **BookService**
   - Depende de: `BookRepository`, `EditorialRepository`
   - Validaciones: Editorial debe existir, ISBN debe ser válido

2. **BookAuthorService**
   - Depende de: `BookRepository`, `AuthorRepository`, `BookService`
   - Gestiona relaciones entre libros y autores

3. **AuthorService**
   - Depende de: `AuthorRepository`
   - Autonomía en operaciones básicas

4. **EditorialService**
   - Depende de: `EditorialRepository`
   - Gestiona editoriales

5. **PrizeService**
   - Depende de: `PrizeRepository`, `AuthorRepository`, `OrganizationRepository`
   - Validaciones complejas para premios

#### **Inyección de Dependencias:**

```java
@Service
public class BookService {
    @Autowired
    BookRepository bookRepository;
    
    @Autowired
    EditorialRepository editorialRepository;
}
```

#### **Transaccionalidad:**

Las transacciones se manejan a nivel de servicio:
- Una operación en un servicio mantiene su propia transacción
- Las dependencias entre servicios no implican transacciones distribuidas
- El acceso a datos es sincrónico

---

### 8. **¿Cómo se gestionan la seguridad y la autenticación dentro de la aplicación?**

#### **Estado Actual de Seguridad:**

**La aplicación NO implementa autenticación ni autorización en tiempo actual.** Esto es apropiado para una aplicación de referencia/educativa.

#### **CORS (Cross-Origin Resource Sharing):**

```java
@Configuration
public class ApplicationConfig {
    @Bean
    WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                    .allowedOrigins("*")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .maxAge(3600);
            }
        };
    }
}
```

**Características:**
- **Origins permitidos**: Todos (`*`)
- **Métodos permitidos**: GET, POST, PUT, DELETE
- **Max Age**: 3600 segundos
- **Nivel de seguridad**: BAJO (apropiado solo para desarrollo)

#### **Recomendaciones para Producción:**

1. **Autenticación OAuth2/JWT**
   ```java
   // Agregar dependencia
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-security</artifactId>
   </dependency>
   ```

2. **CORS Restrictivo**
   ```java
   registry.addMapping("/api/**")
       .allowedOrigins("https://trusted-domain.com")
       .allowedMethods("GET", "POST", "PUT", "DELETE")
       .allowCredentials(true);
   ```

3. **HTTPS/TLS**
   - Obligatorio en producción

4. **Validación de Entrada**
   - Usar `@Valid` con Bean Validation

5. **Encriptación de Contraseñas**
   - BCryptPasswordEncoder

---

### 9. **¿Existen mecanismos de escalabilidad y balanceo de carga?**

#### **Escalabilidad Horizontal - Potencial:**

**La aplicación está preparada para escalabilidad horizontal**, pero no hay mecanismos de balanceo de carga configurados actualmente. La escalabilidad sería posible mediante:

1. **Containerización**
   - El `Dockerfile` permite crear múltiples instancias de la aplicación
   - Cada contenedor sería independiente y sin estado

2. **Diseño Stateless**
   - La aplicación es stateless (no mantiene estado en memoria compartida)
   - Cada solicitud HTTP es independiente
   - Permitiría distribuir tráfico entre múltiples instancias

3. **Base de Datos Compartida**
   - Múltiples instancias pueden acceder a la misma base de datos
   - La persistencia está centralizada, no distribuida

#### **Configuración Recomendada para Escalabilidad:**

Sin evidencia en el código actual, se podría implementar con:

**Opción 1: Docker Swarm**
```bash
docker swarm init
docker service create --replicas 3 --name bookstore-back bookstore-back:latest
```

**Opción 2: Kubernetes**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore-back
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bookstore-back
  template:
    metadata:
      labels:
        app: bookstore-back
    spec:
      containers:
      - name: bookstore-back
        image: bookstore-back:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: bookstore-back-service
spec:
  selector:
    app: bookstore-back
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

**Opción 3: Nginx/HAProxy (Load Balancer)**
```nginx
upstream bookstore {
    server app1:8080;
    server app2:8080;
    server app3:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://bookstore;
    }
}
```

#### **Optimizaciones Implementadas:**

1. **Lazy Loading de Entidades**
   ```java
   @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
   private List<PrizeEntity> prizes = new ArrayList<>();
   ```

2. **Connection Pooling** (implícito en Spring Data JPA)
   - Hibernate gestiona automáticamente el pool de conexiones

3. **Transacciones** 
   - Solo cuando es necesario (`@Transactional`)

#### **Mecanismos NO Implementados (Pero Recomendados para Producción):**

❌ Balanceo de carga (Nginx/HAProxy/Cloud Load Balancer)
❌ Caché distribuida (Redis/Memcached)
❌ Message Queue (RabbitMQ/Kafka/AWS SQS)
❌ CDN para contenido estático
❌ Particionamiento de base de datos (Sharding)
❌ Replicación de base de datos
❌ Auto-scaling (basado en métricas)

---

### 10. **¿Cómo se manejan los errores y la resiliencia del sistema?**

#### **Manejo Centralizado de Excepciones:**

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@ControllerAdvice
public class RestExceptionHandler extends ResponseEntityExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    protected ResponseEntity<Object> handleEntityNotFound(
            EntityNotFoundException ex) {
        ApiError apiError = new ApiError(NOT_FOUND);
        apiError.setMessage(ex.getMessage());
        return buildResponseEntity(apiError);
    }
    
    @ExceptionHandler(IllegalOperationException.class)
    protected ResponseEntity<Object> handleIllegalOperation(
            IllegalOperationException ex) {
        ApiError apiError = new ApiError(PRECONDITION_FAILED);
        apiError.setMessage(ex.getMessage());
        return buildResponseEntity(apiError);
    }
}
```

#### **Excepciones Personalizadas:**

1. **EntityNotFoundException** → HTTP 404 (NOT_FOUND)
   - Cuando una entidad no existe
   - Mensaje: "Entity with id X not found"

2. **IllegalOperationException** → HTTP 412 (PRECONDITION_FAILED)
   - Cuando una operación viola reglas de negocio
   - Ejemplos: ISBN duplicado, Editorial inválida

#### **ApiError Response:**

```java
public class ApiError {
    private HttpStatus status;
    private String message;
    private Long timestamp;
    
    // Getters y setters
}
```

#### **Respuesta de Error:**

```json
{
    "status": 404,
    "message": "Book with id 999 not found",
    "timestamp": 1234567890
}
```

#### **Logging:**

```java
@Slf4j
@Service
public class BookService {
    
    @Transactional
    public BookEntity createBook(BookEntity bookEntity) {
        log.info("Inicia proceso de creación del libro");
        // ... lógica ...
        log.info("Termina proceso de creación del libro");
        return bookRepository.save(bookEntity);
    }
}
```

#### **Validaciones en Servicios:**

```java
public BookEntity createBook(BookEntity bookEntity) 
    throws EntityNotFoundException, IllegalOperationException {
    
    // Validación 1: Editorial es obligatoria
    if (bookEntity.getEditorial() == null)
        throw new IllegalOperationException("Editorial is not valid");
    
    // Validación 2: Editorial existe
    Optional<EditorialEntity> editorialEntity = 
        editorialRepository.findById(bookEntity.getEditorial().getId());
    if (editorialEntity.isEmpty())
        throw new IllegalOperationException("Editorial is not valid");
    
    // Validación 3: ISBN válido
    if (!validateISBN(bookEntity.getIsbn()))
        throw new IllegalOperationException("ISBN is not valid");
    
    // Validación 4: ISBN no duplicado
    if (!bookRepository.findByIsbn(bookEntity.getIsbn()).isEmpty())
        throw new IllegalOperationException("ISBN already exists");
    
    return bookRepository.save(bookEntity);
}
```

#### **Mecanismos No Implementados:**

- **Circuit Breaker**
- **Retry Logic**
- **Timeout Policies**
- **Health Checks**
- **Rate Limiting**

#### **Recomendaciones para Mejorar:**

```java
// Agregar Resilience4j
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>

// Usar @CircuitBreaker, @Retry, @Timeout
@Service
public class BookService {
    @CircuitBreaker(name = "bookService")
    @Retry(name = "bookService")
    public BookEntity getBook(Long id) {
        // ...
    }
}
```

---

### 11. **¿Cómo se entienden las capas de la aplicación y cómo se manejan?**

#### **Arquitectura en Capas (Layered Architecture):**
┌────────────────────────────────────────────────────┐
│          CAPA DE PRESENTACIÓN (REST API)           │
│  Controllers (HTTP Request/Response Handling)      │
│                                                    │
│  BookController, AuthorController, etc.            │
│  - @RestController                                 │
│  - @RequestMapping("/endpoint")                    │
│  - Convierte DTOs ↔ Entidades                      │
└────────────────┬─────────────────────────────────  ┘
                 │ @Autowired
                 ▼
┌────────────────────────────────────────────────────┐
│      CAPA DE LÓGICA DE NEGOCIO (Services)          │
│  Implementa reglas de negocio y orquestación       │
│                                                    │
│  BookService, AuthorService, etc.                  │
│  - @Service                                        │
│  - @Transactional                                  │
│  - Validaciones complejas                          │
│  - Orquestación de operaciones                     │
└────────────────┬─────────────────────────────────┘
                 │ @Autowired
                 ▼
┌────────────────────────────────────────────────────┐
│   CAPA DE ACCESO A DATOS (Repositories)            │
│  Abstracción de la fuente de datos                 │
│                                                    │
│  BookRepository, AuthorRepository, etc.            │
│  - extends JpaRepository<Entity, Long>             │
│  - Consultas a base de datos                       │
│  - Métodos personalizados de búsqueda              │
└────────────────┬─────────────────────────────────┘
                 │ Spring Data JPA
                 ▼
┌────────────────────────────────────────────────────┐
│        CAPA DE PERSISTENCIA (Database)             │
│  H2 (Desarrollo) / PostgreSQL (Producción)         │
│                                                    │
│  Tablas: book_entity, author_entity, etc.          │
│  - SQL Schema                                      │
│  - Datos persistidos                               │
└────────────────────────────────────────────────────┘

```

#### **Características de Cada Capa:**

| Capa | Responsabilidad | Componentes | Tecnología |
|------|-----------------|-------------|-----------|
| **Presentación** | Manejo de solicitudes HTTP | Controllers | Spring Web |
| **Negocio** | Lógica de aplicación | Services | Spring |
| **Datos** | Acceso a BD | Repositories | Spring Data JPA |
| **Persistencia** | Almacenamiento | Database | H2/SQL |

#### **Comunicación Entre Capas:**

1. **Controlador → Servicio**
   ```java
   @RestController
   public class BookController {
       @Autowired
       private BookService bookService;
       
       @GetMapping("/{id}")
       public BookDetailDTO findOne(@PathVariable Long id) {
           BookEntity entity = bookService.getBook(id); // Llamada a servicio
           return modelMapper.map(entity, BookDetailDTO.class);
       }
   }
   ```

2. **Servicio → Repositorio**
   ```java
   @Service
   public class BookService {
       @Autowired
       private BookRepository bookRepository;
       
       public List<BookEntity> getBooks() {
           return bookRepository.findAll(); // Llamada a repositorio
       }
   }
   ```

3. **Transformación de Datos**
   ```java
   // DTO → Entity (entrada)
   BookEntity entity = modelMapper.map(bookDTO, BookEntity.class);
   
   // Entity → DTO (salida)
   BookDetailDTO dto = modelMapper.map(entity, BookDetailDTO.class);
   ```

#### **Ventajas del Diseño en Capas:**

✅ Separación de responsabilidades
✅ Facilita testing de cada capa
✅ Reutilización de servicios
✅ Mantenibilidad
✅ Escalabilidad

#### **Desventajas:**

❌ Código repetitivo (boilerplate)
❌ Puede ser excesivo para aplicaciones pequeñas
❌ Puede afectar rendimiento con múltiples capas

---

### 12. **¿Cómo me puedo comunicar con esta aplicación?: API? mecanismo de comunicación. Si es un api generar el código para entender cuales son los endpoints**

#### **Tipo de Comunicación:**

La aplicación expone una **API RESTful HTTP/HTTPS** como único mecanismo de comunicación.

#### **Configuración Base:**

```properties
# application.properties
server.servlet.context-path=/api
# Contexto base: http://localhost:8080/api
```

#### **Protocolo:**
- **HTTP/HTTPS** (REST)
- **Métodos soportados**: GET, POST, PUT, DELETE
- **Formato de datos**: JSON (application/json)
- **Puerto**: 8080 (por defecto)

---

#### **ENDPOINTS PRINCIPALES:**

##### **1. BOOKS (Libros)**

```java
// GET: Obtener todos los libros
GET /api/books
Headers: Content-Type: application/json

Response: 200 OK
[
  {
    "id": 1,
    "name": "Clean Code",
    "isbn": "978-0132350884",
    "image": "url/image.jpg",
    "publishingDate": "2008-08-01",
    "description": "A book about clean code",
    "editorial": { "id": 1, "name": "Prentice Hall" },
    "reviews": [],
    "authors": []
  }
]

---

// GET: Obtener un libro por ID
GET /api/books/{id}
Example: GET /api/books/1
Headers: Content-Type: application/json

Response: 200 OK
{
  "id": 1,
  "name": "Clean Code",
  "isbn": "978-0132350884",
  "image": "url/image.jpg",
  "publishingDate": "2008-08-01",
  "description": "A book about clean code",
  "editorial": { "id": 1, "name": "Prentice Hall" },
  "reviews": [],
  "authors": []
}

---

// POST: Crear un nuevo libro
POST /api/books
Headers: Content-Type: application/json

Request Body:
{
  "name": "The Clean Coder",
  "isbn": "978-0137081073",
  "image": "url/image2.jpg",
  "publishingDate": "2011-08-01",
  "description": "Professional practices for coding",
  "editorial": { "id": 1 }
}

Response: 201 CREATED
{
  "id": 2,
  "name": "The Clean Coder",
  "isbn": "978-0137081073",
  "image": "url/image2.jpg",
  "publishingDate": "2011-08-01",
  "description": "Professional practices for coding",
  "editorial": { "id": 1 }
}

---

// PUT: Actualizar un libro
PUT /api/books/{id}
Example: PUT /api/books/1
Headers: Content-Type: application/json

Request Body:
{
  "name": "Clean Code (Updated)",
  "isbn": "978-0132350884",
  "image": "url/image_updated.jpg",
  "publishingDate": "2008-08-01",
  "description": "Updated description",
  "editorial": { "id": 1 }
}

Response: 200 OK
{
  "id": 1,
  "name": "Clean Code (Updated)",
  // ... resto de propiedades
}

---

// DELETE: Eliminar un libro
DELETE /api/books/{id}
Example: DELETE /api/books/1
Headers: Content-Type: application/json

Response: 204 NO CONTENT
```

---

##### **2. AUTHORS (Autores)**

```java
// GET: Obtener todos los autores
GET /api/authors
Response: 200 OK
[
  {
    "id": 1,
    "name": "Robert C. Martin",
    "birthDate": "1952-12-17",
    "description": "Software engineer and author",
    "image": "url/author.jpg",
    "books": [],
    "prizes": []
  }
]

---

// GET: Obtener un autor por ID
GET /api/authors/{id}
Example: GET /api/authors/1
Response: 200 OK
{
  "id": 1,
  "name": "Robert C. Martin",
  "birthDate": "1952-12-17",
  "description": "Software engineer and author",
  "image": "url/author.jpg"
}

---

// POST: Crear un nuevo autor
POST /api/authors
Request Body:
{
  "name": "Steve McConnell",
  "birthDate": "1962-06-01",
  "description": "Author of Code Complete",
  "image": "url/author2.jpg"
}

Response: 201 CREATED
{
  "id": 2,
  "name": "Steve McConnell",
  "birthDate": "1962-06-01",
  "description": "Author of Code Complete",
  "image": "url/author2.jpg"
}

---

// PUT: Actualizar un autor
PUT /api/authors/{id}
Example: PUT /api/authors/1
Request Body:
{
  "name": "Robert C. Martin (Updated)",
  "birthDate": "1952-12-17",
  "description": "Legend in software engineering",
  "image": "url/author_updated.jpg"
}

Response: 200 OK

---

// DELETE: Eliminar un autor
DELETE /api/authors/{id}
Example: DELETE /api/authors/1
Response: 204 NO CONTENT
```

---

##### **3. EDITORIALS (Editoriales)**

```java
// GET: Obtener todas las editoriales
GET /api/editorials
Response: 200 OK
[
  {
    "id": 1,
    "name": "Prentice Hall"
  }
]

---

// GET: Obtener una editorial por ID
GET /api/editorials/{id}
Example: GET /api/editorials/1
Response: 200 OK
{
  "id": 1,
  "name": "Prentice Hall",
  "books": []
}

---

// POST: Crear una nueva editorial
POST /api/editorials
Request Body:
{
  "name": "O'Reilly Media"
}

Response: 201 CREATED
{
  "id": 2,
  "name": "O'Reilly Media"
}

---

// PUT: Actualizar una editorial
PUT /api/editorials/{id}
Example: PUT /api/editorials/1
Request Body:
{
  "name": "Prentice Hall (Updated)"
}

Response: 200 OK

---

// DELETE: Eliminar una editorial
DELETE /api/editorials/{id}
Example: DELETE /api/editorials/1
Response: 204 NO CONTENT
```

---

##### **4. REVIEWS (Reseñas)**

```java
// GET: Obtener todas las reseñas
GET /api/reviews
Response: 200 OK
[
  {
    "id": 1,
    "name": "Review Title",
    "description": "Great book!",
    "source": "Goodreads",
    "book": { "id": 1 }
  }
]

---

// POST: Crear una nueva reseña
POST /api/reviews
Request Body:
{
  "name": "Excellent read",
  "description": "Highly recommended",
  "source": "Amazon",
  "book": { "id": 1 }
}

Response: 201 CREATED
{
  "id": 2,
  "name": "Excellent read",
  "description": "Highly recommended",
  "source": "Amazon"
}

---

// PUT: Actualizar una reseña
PUT /api/reviews/{id}
Example: PUT /api/reviews/1
Request Body:
{
  "name": "Updated Review",
  "description": "Still great",
  "source": "Goodreads",
  "book": { "id": 1 }
}

Response: 200 OK

---

// DELETE: Eliminar una reseña
DELETE /api/reviews/{id}
Example: DELETE /api/reviews/1
Response: 204 NO CONTENT
```

---

##### **5. PRIZES (Premios)**

```java
// GET: Obtener todos los premios
GET /api/prizes
Response: 200 OK
[
  {
    "id": 1,
    "name": "Martin Fowler Award",
    "description": "For contributions to software design",
    "premiationDate": "2023-06-15",
    "author": { "id": 1 },
    "organization": { "id": 1 }
  }
]

---

// POST: Crear un nuevo premio
POST /api/prizes
Request Body:
{
  "name": "Tech Innovation Award",
  "description": "For software innovation",
  "premiationDate": "2023-07-20",
  "author": { "id": 1 },
  "organization": { "id": 1 }
}

Response: 201 CREATED
{
  "id": 2,
  "name": "Tech Innovation Award",
  // ... resto
}

---

// PUT: Actualizar un premio
PUT /api/prizes/{id}
Example: PUT /api/prizes/1
Request Body:
{
  "name": "Martin Fowler Award (Updated)",
  "description": "Updated description",
  "premiationDate": "2023-06-15",
  "author": { "id": 1 },
  "organization": { "id": 1 }
}

Response: 200 OK

---

// DELETE: Eliminar un premio
DELETE /api/prizes/{id}
Example: DELETE /api/prizes/1
Response: 204 NO CONTENT
```

---

##### **6. ORGANIZATIONS (Organizaciones)**

```java
// GET: Obtener todas las organizaciones
GET /api/organizations
Response: 200 OK
[
  {
    "id": 1,
    "name": "IEEE",
    "tipo": 1
  }
]

---

// POST: Crear una nueva organización
POST /api/organizations
Request Body:
{
  "name": "ACM",
  "tipo": 2
}

Response: 201 CREATED
{
  "id": 2,
  "name": "ACM",
  "tipo": 2
}

---

// PUT: Actualizar una organización
PUT /api/organizations/{id}
Example: PUT /api/organizations/1
Request Body:
{
  "name": "IEEE (Updated)",
  "tipo": 1
}

Response: 200 OK

---

// DELETE: Eliminar una organización
DELETE /api/organizations/{id}
Example: DELETE /api/organizations/1
Response: 204 NO CONTENT
```

---

##### **7. RELACIONES - BOOK-AUTHOR (Relación Muchos-a-Muchos)**

```java
// POST: Asociar un autor a un libro
POST /api/books/{bookId}/authors/{authorId}
Example: POST /api/books/1/authors/2
Response: 200 OK
{
  "id": 2,
  "name": "Steve McConnell",
  "birthDate": "1962-06-01",
  "description": "Author of Code Complete",
  "image": "url/author2.jpg"
}

---

// GET: Obtener todos los autores de un libro
GET /api/books/{bookId}/authors
Example: GET /api/books/1/authors
Response: 200 OK
[
  {
    "id": 1,
    "name": "Robert C. Martin",
    // ... resto
  },
  {
    "id": 2,
    "name": "Steve McConnell",
    // ... resto
  }
]

---

// GET: Obtener un autor específico de un libro
GET /api/books/{bookId}/authors/{authorId}
Example: GET /api/books/1/authors/1
Response: 200 OK
{
  "id": 1,
  "name": "Robert C. Martin",
  // ... resto
}

---

// DELETE: Remover un autor de un libro
DELETE /api/books/{bookId}/authors/{authorId}
Example: DELETE /api/books/1/authors/2
Response: 204 NO CONTENT
```

---

##### **8. RELACIONES - AUTHOR-BOOK (Relación Inversa)**

```java
// POST: Asociar un libro a un autor
POST /api/authors/{authorId}/books/{bookId}
Example: POST /api/authors/1/books/2
Response: 200 OK
{
  "id": 2,
  "name": "The Clean Coder",
  // ... resto
}

---

// GET: Obtener todos los libros de un autor
GET /api/authors/{authorId}/books
Example: GET /api/authors/1/books
Response: 200 OK
[
  {
    "id": 1,
    "name": "Clean Code",
    // ... resto
  }
]

---

// DELETE: Remover un libro de un autor
DELETE /api/authors/{authorId}/books/{bookId}
Example: DELETE /api/authors/1/books/1
Response: 204 NO CONTENT
```

---

##### **9. RELACIONES - BOOK-EDITORIAL**

```java
// POST: Asociar una editorial a un libro
POST /api/books/{bookId}/editorials/{editorialId}
Response: 200 OK

---

// GET: Obtener la editorial de un libro
GET /api/books/{bookId}/editorials
Response: 200 OK
{
  "id": 1,
  "name": "Prentice Hall"
}

---

// DELETE: Remover la editorial de un libro
DELETE /api/books/{bookId}/editorials
Response: 204 NO CONTENT
```

---

##### **10. RELACIONES - EDITORIAL-BOOK**

```java
// GET: Obtener todos los libros de una editorial
GET /api/editorials/{editorialId}/books
Example: GET /api/editorials/1/books
Response: 200 OK
[
  {
    "id": 1,
    "name": "Clean Code",
    // ... resto
  },
  {
    "id": 2,
    "name": "The Clean Coder",
    // ... resto
  }
]

---

// POST: Asociar un libro a una editorial
POST /api/editorials/{editorialId}/books/{bookId}
Response: 200 OK

---

// DELETE: Remover un libro de una editorial
DELETE /api/editorials/{editorialId}/books/{bookId}
Response: 204 NO CONTENT
```

---

##### **11. RELACIONES - AUTHOR-PRIZE**

```java
// GET: Obtener todos los premios de un autor
GET /api/authors/{authorId}/prizes
Example: GET /api/authors/1/prizes
Response: 200 OK
[
  {
    "id": 1,
    "name": "Martin Fowler Award",
    "description": "For contributions",
    "premiationDate": "2023-06-15",
    "organization": { "id": 1 }
  }
]

---

// POST: Asociar un premio a un autor
POST /api/authors/{authorId}/prizes/{prizeId}
Response: 200 OK

---

// DELETE: Remover un premio de un autor
DELETE /api/authors/{authorId}/prizes/{prizeId}
Response: 204 NO CONTENT
```

---

#### **MANEJO DE ERRORES:**

```java
// Respuesta de error 404
GET /api/books/999
Response: 404 NOT FOUND
{
  "status": 404,
  "message": "Book with id 999 not found",
  "timestamp": 1697564800000
}

---

// Respuesta de error 412
POST /api/books
Request Body:
{
  "name": "Clean Code",
  "isbn": "978-0132350884",  // ISBN duplicado
  "editorial": { "id": 1 }
}

Response: 412 PRECONDITION_FAILED
{
  "status": 412,
  "message": "ISBN already exists",
  "timestamp": 1697564800000
}
```

---

#### **CÓDIGO CLIENTE EJEMPLO (JavaScript/Fetch):**

```javascript
// 1. Obtener todos los libros
async function getBooks() {
  const response = await fetch('http://localhost:8080/api/books');
  const data = await response.json();
  console.log(data);
}

// 2. Obtener un libro específico
async function getBook(id) {
  const response = await fetch(`http://localhost:8080/api/books/${id}`);
  const data = await response.json();
  console.log(data);
}

// 3. Crear un nuevo libro
async function createBook() {
  const response = await fetch('http://localhost:8080/api/books', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: 'Clean Code',
      isbn: '978-0132350884',
      image: 'url/image.jpg',
      publishingDate: '2008-08-01',
      description: 'A book about clean code',
      editorial: { id: 1 }
    })
  });
  const data = await response.json();
  console.log(data);
}

// 4. Actualizar un libro
async function updateBook(id) {
  const response = await fetch(`http://localhost:8080/api/books/${id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: 'Clean Code (Updated)',
      isbn: '978-0132350884',
      image: 'url/image_updated.jpg',
      publishingDate: '2008-08-01',
      description: 'Updated description',
      editorial: { id: 1 }
    })
  });
  const data = await response.json();
  console.log(data);
}

// 5. Eliminar un libro
async function deleteBook(id) {
  const response = await fetch(`http://localhost:8080/api/books/${id}`, {
    method: 'DELETE'
  });
  if (response.status === 204) {
    console.log('Book deleted successfully');
  }
}

// 6. Asociar un autor a un libro
async function addAuthorToBook(bookId, authorId) {
  const response = await fetch(
    `http://localhost:8080/api/books/${bookId}/authors/${authorId}`,
    { method: 'POST' }
  );
  const data = await response.json();
  console.log(data);
}

// 7. Obtener autores de un libro
async function getBookAuthors(bookId) {
  const response = await fetch(
    `http://localhost:8080/api/books/${bookId}/authors`
  );
  const data = await response.json();
  console.log(data);
}
```

---

#### **CÓDIGO CLIENTE EJEMPLO (Python/Requests):**

```python
import requests

BASE_URL = 'http://localhost:8080/api'

# 1. Obtener todos los libros
def get_books():
    response = requests.get(f'{BASE_URL}/books')
    print(response.json())

# 2. Obtener un libro específico
def get_book(book_id):
    response = requests.get(f'{BASE_URL}/books/{book_id}')
    print(response.json())

# 3. Crear un nuevo libro
def create_book():
    book = {
        'name': 'Clean Code',
        'isbn': '978-0132350884',
        'image': 'url/image.jpg',
        'publishingDate': '2008-08-01',
        'description': 'A book about clean code',
        'editorial': {'id': 1}
    }
    response = requests.post(f'{BASE_URL}/books', json=book)
    print(response.json())

# 4. Actualizar un libro
def update_book(book_id):
    book = {
        'name': 'Clean Code (Updated)',
        'isbn': '978-0132350884',
        'image': 'url/image_updated.jpg',
        'publishingDate': '2008-08-01',
        'description': 'Updated description',
        'editorial': {'id': 1}
    }
    response = requests.put(f'{BASE_URL}/books/{book_id}', json=book)
    print(response.json())

# 5. Eliminar un libro
def delete_book(book_id):
    response = requests.delete(f'{BASE_URL}/books/{book_id}')
    print(f'Status: {response.status_code}')

# 6. Asociar un autor a un libro
def add_author_to_book(book_id, author_id):
    response = requests.post(f'{BASE_URL}/books/{book_id}/authors/{author_id}')
    print(response.json())

# 7. Obtener autores de un libro
def get_book_authors(book_id):
    response = requests.get(f'{BASE_URL}/books/{book_id}/authors')
    print(response.json())

# Ejecutar ejemplos
if __name__ == '__main__':
    get_books()
    create_book()
    get_book(1)
    update_book(1)
    add_author_to_book(1, 1)
    get_book_authors(1)
    delete_book(1)
```

---

#### **CÓDIGO CLIENTE EJEMPLO (cURL):**

```bash
# 1. Obtener todos los libros
curl -X GET http://localhost:8080/api/books \
  -H "Content-Type: application/json"

# 2. Obtener un libro específico
curl -X GET http://localhost:8080/api/books/1 \
  -H "Content-Type: application/json"

# 3. Crear un nuevo libro
curl -X POST http://localhost:8080/api/books \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Clean Code",
    "isbn": "978-0132350884",
    "image": "url/image.jpg",
    "publishingDate": "2008-08-01",
    "description": "A book about clean code",
    "editorial": {"id": 1}
  }'

# 4. Actualizar un libro
curl -X PUT http://localhost:8080/api/books/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Clean Code (Updated)",
    "isbn": "978-0132350884",
    "image": "url/image_updated.jpg",
    "publishingDate": "2008-08-01",
    "description": "Updated description",
    "editorial": {"id": 1}
  }'

# 5. Eliminar un libro
curl -X DELETE http://localhost:8080/api/books/1 \
  -H "Content-Type: application/json"

# 6. Asociar un autor a un libro
curl -X POST http://localhost:8080/api/books/1/authors/1 \
  -H "Content-Type: application/json"

# 7. Obtener autores de un libro
curl -X GET http://localhost:8080/api/books/1/authors \
  -H "Content-Type: application/json"
```

---

#### **RESUMEN DE ENDPOINTS:**

| Recurso | Método | Endpoint | Descripción |
|---------|--------|----------|-------------|
| **Books** | GET | `/api/books` | Obtener todos |
| | GET | `/api/books/{id}` | Obtener uno |
| | POST | `/api/books` | Crear |
| | PUT | `/api/books/{id}` | Actualizar |
| | DELETE | `/api/books/{id}` | Eliminar |
| **Authors** | GET | `/api/authors` | Obtener todos |
| | GET | `/api/authors/{id}` | Obtener uno |
| | POST | `/api/authors` | Crear |
| | PUT | `/api/authors/{id}` | Actualizar |
| | DELETE | `/api/authors/{id}` | Eliminar |
| **Editorials** | GET | `/api/editorials` | Obtener todos |
| | GET | `/api/editorials/{id}` | Obtener uno |
| | POST | `/api/editorials` | Crear |
| | PUT | `/api/editorials/{id}` | Actualizar |
| | DELETE | `/api/editorials/{id}` | Eliminar |
| **Reviews** | GET | `/api/reviews` | Obtener todos |
| | POST | `/api/reviews` | Crear |
| | PUT | `/api/reviews/{id}` | Actualizar |
| | DELETE | `/api/reviews/{id}` | Eliminar |
| **Prizes** | GET | `/api/prizes` | Obtener todos |
| | POST | `/api/prizes` | Crear |
| | PUT | `/api/prizes/{id}` | Actualizar |
| | DELETE | `/api/prizes/{id}` | Eliminar |
| **Organizations** | GET | `/api/organizations` | Obtener todos |
| | POST | `/api/organizations` | Crear |
| | PUT | `/api/organizations/{id}` | Actualizar |
| | DELETE | `/api/organizations/{id}` | Eliminar |
| **Relaciones** | POST | `/api/books/{id}/authors/{id}` | Asociar autor |
| | GET | `/api/books/{id}/authors` | Obtener autores |
| | DELETE | `/api/books/{id}/authors/{id}` | Remover autor |
| | POST | `/api/authors/{id}/books/{id}` | Asociar libro |
| | GET | `/api/authors/{id}/books` | Obtener libros |
| | DELETE | `/api/authors/{id}/books/{id}` | Remover libro |

---

## Conclusión

La aplicación **Bookstore Backend** es una aplicación Spring Boot monolítica, educativa y bien estructurada que implementa:

✅ **Arquitectura en Capas**: Separación clara de responsabilidades
✅ **Patrones de Diseño**: Repository, DTO, Service Layer, Exception Handling
✅ **REST API**: Comunicación HTTP/JSON estándar
✅ **Base de Datos**: JPA/Hibernate con H2 (escalable a bases de datos relacionales)
✅ **Containerización Preparada**: Dockerfile para despliegue en contenedores
✅ **CI/CD**: Integración continua con Jenkins y SonarQube (utiliza Docker en el pipeline)
✅ **Testing**: Cobertura con JaCoCo
✅ **Diseño Stateless**: Preparada para escalabilidad horizontal

**Clarificaciones importantes:**
- La aplicación **está lista para ser containerizada con Docker**, pero no hay evidencia de despliegue actual en producción.
- El pipeline **CI/CD utiliza contenedores Docker** para construcción y testing.
- **No hay evidencia** de orquestación en producción (Kubernetes, Docker Swarm, etc.).
- Aunque la aplicación no implementa autenticación ni autorización actualmente, está diseñada de manera que estos componentes pueden agregarse fácilmente.
- La escalabilidad horizontal es **posible pero no está configurada** debido al diseño stateless y la preparación de containerización.
