# 📚 ARCN-Peach — Sistema de Biblioteca Digital

Sistema de gestión de biblioteca implementado como **arquitectura de microservicios** con Java 21, Spring Boot, PostgreSQL y RabbitMQ. Cada servicio es dueño exclusivo de su bounded context, se comunica de forma asíncrona mediante eventos y sigue los principios de **Clean Architecture** y **Domain-Driven Design**.

---

## 🗂️ Repositorios

| Repositorio | Bounded Context | Puerto | Estado |
|---|---|---|---|
| [library-catalog](https://github.com/ARCN-Peach/library-catalog) | Gestión de Catálogo | `8082` | ✅ Activo |
| [library-user](https://github.com/ARCN-Peach/library-user) | Gestión de Usuarios | `8081` | ✅ Activo |
| [library-rental](https://github.com/ARCN-Peach/library-rental) | Préstamos y Devoluciones | `8083` | ✅ Activo |
| [library-fine](https://github.com/ARCN-Peach/library-fine) | Gestión de Multas | `8084` | ✅ Activo |

---

## 🏗️ Arquitectura del sistema

```
                        ┌──────────────────────────────────────────────────┐
                        │              RabbitMQ  (Event Bus)                │
                        └────┬──────────────┬──────────────┬───────────────┘
                             │              │              │
              ┌──────────────┘    ┌─────────┘    ┌────────┘
              ▼                   ▼               ▼
┌─────────────────────┐  ┌───────────────┐  ┌──────────────────┐
│   library-catalog   │  │ library-user  │  │  library-fine    │
│   puerto 8082       │  │ puerto 8081   │  │  puerto 8084     │
│                     │  │               │  │                  │
│ Gestiona libros,    │  │ Auth JWT,     │  │ Genera y cobra   │
│ stock y búsquedas   │  │ roles, perfil │  │ multas por retraso│
└──────────┬──────────┘  └───────────────┘  └────────┬─────────┘
           │                    ▲                     ▲
           │                    │                     │
           ▼                    │                     │
┌─────────────────────┐         │                     │
│   library-rental    │─────────┘─────────────────────┘
│   puerto 8083       │  publica: BookReturnedEvent,
│                     │          BookLentEvent,
│ Préstamos y         │          RentalOverdueEvent
│ devoluciones        │
└─────────────────────┘
```

### Principios transversales

- **Clean Architecture** en todos los servicios: Interfaces → Application → Domain ← Infrastructure
- **Domain-Driven Design**: cada servicio es el único dueño de su agregado raíz
- **Outbox Pattern**: escritura de entidad y evento en la misma transacción; publicación asíncrona confiable
- **Idempotencia**: guardias en base de datos y lógica de negocio para evitar duplicados
- **Dead-Letter Queues (DLQ)**: mensajes fallidos se enrutan automáticamente para reintento o análisis

---

## 📦 Microservicios detallados

### 🔵 library-catalog — Gestión de Catálogo
> [Ver repositorio](https://github.com/ARCN-Peach/library-catalog)

Registra, actualiza, retira y permite buscar libros. Mantiene el **stock disponible** de cada título reaccionando a eventos de préstamos y devoluciones.

**Agregado raíz:** `Book`

**Stack:** Java 21 · Spring Boot 3.4 · PostgreSQL · RabbitMQ · Flyway · ArchUnit · Testcontainers · Swagger/OpenAPI

| Elemento | Detalle |
|---|---|
| Eventos que publica | `BookRegisteredEvent`, `BookUpdatedEvent`, `BookRetiredEvent` |
| Eventos que consume | `BookLentEvent`, `BookReturnedEvent` (de `library-rental`) |
| Políticas | Solo libros `PUBLISHED` aparecen en búsquedas · stock no puede ser negativo |

**API REST:**

| Método | Endpoint | Descripción |
|---|---|---|
| `POST` | `/books` | Registrar un libro |
| `GET` | `/books/{id}` | Obtener libro por ID |
| `PUT` | `/books/{id}` | Actualizar metadata |
| `DELETE` | `/books/{id}` | Retirar libro del catálogo |
| `GET` | `/books/search` | Buscar por título, autor y/o categoría |

Búsqueda soporta parámetros: `title`, `author`, `category`, `page` (def. 0), `pageSize` (def. 20, máx 100).

```
http://localhost:8082/swagger-ui.html
http://localhost:8082/actuator/health
```

---

### 🟢 library-user — Gestión de Usuarios
> [Ver repositorio](https://github.com/ARCN-Peach/library-user)

Registro y autenticación de lectores con **JWT stateless**. Gestiona perfiles, bloqueo/desbloqueo de cuentas y reacciona a eventos de multas.

**Agregado raíz:** `User`

**Stack:** Java 21 · Spring Boot 3.4 · PostgreSQL · RabbitMQ · Flyway · JWT · SonarCloud

| Elemento | Detalle |
|---|---|
| Eventos que publica | `UserRegisteredEvent`, `UserBlockedEvent` |
| Eventos que consume | `FineGeneratedEvent` (bloquea usuario), `FinePaidEvent` (desbloquea usuario) |
| Políticas | Email único · solo `READER` puede registrarse públicamente · usuarios bloqueados no autentican |

**API REST:**

| Método | Endpoint | Descripción | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Registrar lector | Público |
| `POST` | `/api/v1/auth/login` | Iniciar sesión | Público |
| `POST` | `/api/v1/auth/refresh` | Renovar tokens | Público |
| `GET` | `/api/v1/users/me` | Consultar perfil propio | JWT |
| `PATCH` | `/api/v1/users/me` | Actualizar perfil propio | JWT |
| `GET` | `/api/v1/users/{userId}` | Consultar usuario por ID | JWT + `LIBRARIAN` |
| `PATCH` | `/api/v1/users/{userId}/status` | Bloquear / desbloquear | JWT + `LIBRARIAN` |

```
http://localhost:8081/swagger-ui.html
http://localhost:8081/actuator/health
```

---

### 🟠 library-rental — Préstamos y Devoluciones
> [Ver repositorio](https://github.com/ARCN-Peach/library-rental)

Gestiona el ciclo de vida completo de un préstamo: creación, devolución y consulta. Publica eventos que desencadenan actualizaciones de stock en `library-catalog` y cálculo de multas en `library-fine`.

**Agregado raíz:** `Rental`

**Stack:** Java 21 · Spring Boot 3.4 · PostgreSQL · RabbitMQ · Flyway

| Elemento | Detalle |
|---|---|
| Eventos que publica | `BookLentEvent`, `BookReturnedEvent`, `RentalOverdueEvent` |
| Eventos que consume | — |

**API REST:**

| Método | Endpoint | Descripción |
|---|---|---|
| `POST` | `/api/v1/rentals` | Crear préstamo |
| `PUT` | `/api/v1/rentals/{rentalId}/return` | Registrar devolución |
| `GET` | `/api/v1/rentals/{rentalId}` | Consultar préstamo |
| `GET` | `/api/v1/rentals/users/{userId}` | Listar préstamos de un usuario |
| `GET` | `/api/v1/rentals/overdue` | Listar préstamos vencidos |

```
http://localhost:8083/actuator/health
http://localhost:8083/actuator/health/liveness
```

---

### 🔴 library-fine — Gestión de Multas
> [Ver repositorio](https://github.com/ARCN-Peach/library-fine)

Calcula, registra y cobra multas por devolución tardía. Se basa completamente en eventos: nunca llama directamente a `library-rental`.

**Agregado raíz:** `Fine`

**Stack:** Java 21 · Spring Boot 3.2 · PostgreSQL · RabbitMQ · Flyway · ArchUnit

| Elemento | Detalle |
|---|---|
| Eventos que publica | `FineGeneratedEvent`, `FinePaidEvent` |
| Eventos que consume | `BookReturnedEvent`, `RentalOverdueEvent` (de `library-rental`) |
| Políticas | 1 USD/día de atraso · una multa por préstamo (idempotente) · solo `PENDING` puede pagarse |

**API REST:**

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/fines/{fineId}` | Obtener multa por ID |
| `POST` | `/fines/{fineId}/pay` | Pagar una multa pendiente |
| `GET` | `/fines?userId=&onlyPending=` | Listar multas de un usuario |

```
http://localhost:8084/swagger-ui.html
http://localhost:8084/actuator/health
```

---

## 🔄 Flujo de eventos entre servicios

```
 Cliente                library-rental         library-catalog       library-fine         library-user
    │                        │                       │                    │                    │
    │── POST /rentals ───────►│                       │                    │                    │
    │                        │── BookLentEvent ──────►│ (decrementa stock) │                    │
    │                        │                       │                    │                    │
    │── PUT /return ─────────►│                       │                    │                    │
    │                        │── BookReturnedEvent ──►│ (incrementa stock) │                    │
    │                        │── BookReturnedEvent ──────────────────────►│                    │
    │                        │                       │  GenerateFineUseCase│                    │
    │                        │                       │  (calcula días atraso│                   │
    │                        │                       │  1 USD/día)         │                    │
    │                        │── RentalOverdueEvent ─────────────────────►│                    │
    │                        │                       │                    │                    │
    │                        │                       │                    │── FineGeneratedEvent►│
    │                        │                       │                    │ (bloquea usuario)   │
    │                        │                       │                    │                    │
    │── POST /fines/{id}/pay ────────────────────────────────────────────►│                    │
    │                        │                       │                    │── FinePaidEvent ───►│
    │                        │                       │                    │  (desbloquea usuario│
```

---

## 🛠️ Stack tecnológico

| Tecnología | Versión | Uso |
|---|---|---|
| **Java** | 21 | Lenguaje principal |
| **Spring Boot** | 3.2 – 3.4 | Framework base |
| **Spring Security + JWT** | — | Autenticación stateless (`library-user`) |
| **Spring Data JPA** | — | Persistencia ORM |
| **Spring AMQP** | — | Mensajería con RabbitMQ |
| **PostgreSQL** | — | Base de datos relacional |
| **RabbitMQ** | — | Message broker (Direct exchanges + DLQ) |
| **Flyway** | — | Migraciones de esquema |
| **Springdoc OpenAPI** | 2.x | Swagger UI automático |
| **Spring Actuator** | — | Health checks y métricas |
| **ArchUnit** | 1.3 | Verificación de reglas de arquitectura |
| **Testcontainers** | 1.20 | Tests de integración con contenedores reales |
| **JaCoCo** | 0.8 | Cobertura de tests (umbral mínimo 60–80%) |
| **SonarCloud** | — | Análisis estático de calidad |
| **Docker + Compose** | — | Empaquetado y despliegue local |
| **Railway** | — | Plataforma de despliegue en la nube |
| **Maven** | 3.9+ | Build tool |

---

## 🚀 Levantar el sistema completo en local

Cada servicio puede iniciarse de forma independiente. El orden recomendado es:

```bash
# 1. Catálogo (infraestructura base, no depende de eventos de entrada al inicio)
cd library-catalog && docker compose up --build -d

# 2. Usuarios (necesita RabbitMQ para eventos de multas)
cd library-user && docker compose up --build -d

# 3. Préstamos
cd library-rental && docker compose up --build -d

# 4. Multas (consume eventos de library-rental)
cd library-fine && docker compose up --build -d
```

> Cada `docker-compose.yml` levanta su propia instancia de PostgreSQL y RabbitMQ. Para un entorno compartido, configura un RabbitMQ central y apunta todos los servicios al mismo host mediante variables de entorno.

### URLs de acceso rápido

| Servicio | API | Swagger UI | Actuator | RabbitMQ UI | PostgreSQL |
|---|---|---|---|---|---|
| library-catalog | :8082 | [:8082/swagger-ui.html](http://localhost:8082/swagger-ui.html) | [:8082/actuator/health](http://localhost:8082/actuator/health) | :15672 | :5432 |
| library-user | :8081 | [:8081/swagger-ui.html](http://localhost:8081/swagger-ui.html) | [:8081/actuator/health](http://localhost:8081/actuator/health) | :15673 | :5433 |
| library-rental | :8083 | — | [:8083/actuator/health](http://localhost:8083/actuator/health) | :15672 | :5432 |
| library-fine | :8084 | [:8084/swagger-ui.html](http://localhost:8084/swagger-ui.html) | [:8084/actuator/health](http://localhost:8084/actuator/health) | :15672 | :5432 |

RabbitMQ Management: `guest / guest`

---

## 🧪 Tests y calidad

Todos los servicios incluyen tests que pueden ejecutarse sin infraestructura externa:

```bash
# Tests unitarios
mvn test

# Build completo + reporte de cobertura JaCoCo
mvn verify
```

### Capas de tests

| Capa | Tipo | Qué verifica |
|---|---|---|
| **Domain** | Unit | Invariantes del agregado, Value Objects, servicios de dominio |
| **Application** | Unit | Use cases con repositorios in-memory (sin mocks ni Spring) |
| **Architecture** | ArchUnit | Dependencias entre capas (Clean Architecture) |
| **Integration** | Testcontainers | Persistencia real con PostgreSQL y RabbitMQ en contenedores |

### Reglas de arquitectura verificadas con ArchUnit

1. El dominio no depende de Spring
2. El dominio no depende de infrastructure ni de interfaces
3. La capa application no depende de infrastructure ni de interfaces
4. Las interfaces no acceden directamente a la capa de persistencia
5. La arquitectura por capas se respeta globalmente

### Calidad de código

- **JaCoCo**: cobertura mínima del **60–80%** enforced en `mvn verify`
- **SonarCloud**: Quality Gate activo en `library-user` [![Quality gate](https://sonarcloud.io/api/project_badges/quality_gate?project=ARCN-Peach_library-user&token=2fe2b7e5af4cd0f3728ffb39388a9f3374bdd360)](https://sonarcloud.io/summary/new_code?id=ARCN-Peach_library-user)

---

## 🔐 Seguridad

- **JWT stateless** en `library-user`: sin sesiones en servidor
- Endpoints públicos: `/auth/register`, `/auth/login`, `/auth/refresh`, `/actuator/health/**`
- Endpoints de administración requieren rol `LIBRARIAN`
- Errores devueltos en formato **RFC 9457 ProblemDetail**
- Refresh tokens almacenados como **hash** en base de datos

### Errores estándar

| HTTP | Causa |
|---|---|
| `400 Bad Request` | Argumento inválido o fallo de validación de request |
| `401 Unauthorized` | Token JWT ausente, expirado o inválido |
| `404 Not Found` | Recurso no encontrado |
| `409 Conflict` | Violación de regla de negocio (ej.: multa ya pagada, email duplicado) |

---

## 🗄️ Modelo de base de datos (resumen)

Cada servicio gestiona su propia base de datos. Ningún servicio accede directamente a la BD de otro.

### library-catalog
```sql
books (id, title, author_first_name, author_last_name,
       category, isbn, status, total_copies, available_stock)
outbox_events (id, event_type, payload, occurred_at,
               correlation_id, published, published_at)
```

### library-user
```sql
users (id, name, email [UNIQUE], password_hash, role, status, created_at)
refresh_tokens (id, user_id, token_hash [UNIQUE], expires_at, revoked)
outbox_events (id, event_type, payload, status, occurred_at, published_at)
-- INDEX: (status, id) para polling eficiente
```

### library-fine
```sql
fines (fine_id, rental_id [UNIQUE], user_id, amount,
       currency, status, generated_at, paid_at)
-- rental_id UNIQUE garantiza idempotencia a nivel DB
outbox_events (event_id, event_type, routing_key, exchange,
               payload, occurred_at, published, published_at)
-- INDEX parcial: WHERE published = FALSE para polling eficiente
```

---

## 📨 Mensajería RabbitMQ (mapa completo)

### Exchanges

| Exchange | Tipo | Dueño | Propósito |
|---|---|---|---|
| `catalog` | Direct | library-catalog | Publica eventos del catálogo |
| `rental` | Direct | library-rental | Publica eventos de préstamos |
| `fine` | Direct | library-fine | Publica eventos de multas |
| `library.user.exchange` | — | library-user | Publica eventos de usuarios |
| `fine.dlx` / `catalog.dlx` | Direct | — | Dead-letter para mensajes fallidos |

### Colas y routing keys

| Cola | Routing key | Consumidor |
|---|---|---|
| `catalog.rental.book-lent` | `rental.book.lent.v1` | library-catalog (decrementa stock) |
| `catalog.rental.book-returned` | `rental.book.returned.v1` | library-catalog (incrementa stock) |
| `fine.book-returned` | `rental.rental.book_returned.v1` | library-fine (genera multa) |
| `fine.rental-overdue` | `rental.rental.rental_overdue.v1` | library-fine (genera multa) |
| `user.fine-generated` | `fine.fine.fine_generated.v1` | library-user (bloquea usuario) |
| `user.fine-paid` | `fine.fine.fine_paid.v1` | library-user (desbloquea usuario) |

### Outbox Pattern (todos los servicios)

```
1. INSERT INTO <entidad>        ─┐
2. INSERT INTO outbox_events    ─┤  misma transacción → atómico
3. COMMIT                       ─┘

(scheduler cada 5 segundos)
4. rabbitMQ.send(event)    → si falla, outbox queda published=false
5. UPDATE published = true → reintento automático en el siguiente ciclo
```

---

## ⚙️ Variables de entorno principales

| Variable | Servicios | Descripción |
|---|---|---|
| `DB_HOST` / `DB_URL` | todos | Host o URL JDBC de PostgreSQL |
| `DB_PORT` | catalog, fine | Puerto PostgreSQL |
| `DB_NAME` | catalog, fine | Nombre de base de datos |
| `DB_USER` / `DB_PASSWORD` | todos | Credenciales PostgreSQL |
| `RABBITMQ_HOST` | todos | Host RabbitMQ |
| `RABBITMQ_PORT` | todos | Puerto AMQP (default `5672`) |
| `RABBITMQ_USER` / `RABBITMQ_PASSWORD` | todos | Credenciales RabbitMQ |
| `SERVER_PORT` / `PORT` | todos | Puerto HTTP (Railway inyecta `PORT`) |
| `JWT_SECRET` | library-user | Secreto de firma JWT (≥ 32 chars en producción) |
| `OUTBOX_POLL_INTERVAL_MS` | catalog, fine | Intervalo del scheduler de outbox (ms) |
| `BOOTSTRAP_LIBRARIAN_ENABLED` | library-user | Crea usuario librarian inicial al arrancar |

---

## ☁️ Despliegue en Railway

Todos los servicios están preparados para desplegarse en [Railway](https://railway.app) con PostgreSQL y RabbitMQ administrados.

**Checklist de despliegue:**
1. Configurar variables de entorno de base de datos y RabbitMQ
2. Asegurarse de que el servicio use `PORT` dinámico (`server.port=${PORT:808x}`)
3. Health check apuntando a `/actuator/health/liveness`
4. Permisos de Flyway en PostgreSQL externo:

```sql
GRANT USAGE ON SCHEMA public TO <app_user>;
GRANT CREATE ON SCHEMA public TO <app_user>;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO <app_user>;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO <app_user>;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO <app_user>;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON SEQUENCES TO <app_user>;
```

---

## 🔧 Troubleshooting

| Síntoma | Causa probable | Acción |
|---|---|---|
| Servicio no arranca | Flyway falla al conectar | Verificar variables `DB_*` y permisos de schema |
| `401` en endpoints privados | JWT ausente o expirado | Reautenticar y enviar `Bearer <token>` |
| Usuario no se desbloquea tras pago | Evento `fine_paid` no llega | Revisar cola `user.fine-paid` y bindings en RabbitMQ |
| Stock no se actualiza | Evento `book_returned/lent` no llega | Revisar colas `catalog.rental.*` y bindings |
| Multa duplicada | Falta idempotencia | `rental_id UNIQUE` en tabla `fines` previene duplicados |
| Healthcheck falla en Railway | Puerto incorrecto | Verificar `PORT` y endpoint `/actuator/health/liveness` |

---

## 👥 Equipo

**ARCN-Peach** · Organización de desarrollo de software académico.

> 💡 Para contribuir a cualquier repositorio, abre un issue o pull request directamente en el repositorio correspondiente.
