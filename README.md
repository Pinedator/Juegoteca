# Juegoteca

Aplicación web tipo [Letterboxd](https://letterboxd.com), pero para **videojuegos**. Cada usuario puede buscar juegos, marcar en qué estado los tiene, puntuarlos y escribir una reseña.

> Proyecto personal de aprendizaje de desarrollo backend con Java y Spring Boot.
> **Estado actual:** entorno preparado y modelo de datos diseñado. Las entidades JPA y la API REST están pendientes de implementar.

---

## Funcionalidades

### Versión 1 (MVP)

- Registro e inicio de sesión de usuarios.
- Búsqueda de videojuegos a través de la API de IGDB.
- Añadir un juego a tu colección con un estado:
  - `WANT_TO_PLAY`: quiero jugarlo
  - `PLAYING`: jugando
  - `FINISHED`: terminado
  - `ABANDONED`: abandonado
- Puntuar un juego con una nota **entera de 1 a 10**.
- Escribir una reseña, **una por juego y usuario**, que se puede editar después.
- Guardar la **fecha en la que se jugó** el juego, porque las actualizaciones pueden cambiar la experiencia.

### Para versiones posteriores

- Listas personalizadas de juegos.
- Seguir a otros usuarios.
- Feed de actividad.

---

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Lenguaje | Java 25 |
| Framework | Spring Boot 4.1.1 (Spring Web, Spring Data JPA, Validation, DevTools) |
| Build | Maven |
| Base de datos | PostgreSQL 16 en Docker |
| ORM | Hibernate 7 (vía Spring Data JPA) |
| Datos de juegos | API de [IGDB](https://api-docs.igdb.com) (pendiente de integrar) |
| Seguridad | Spring Security + JWT (pendiente) |
| Frontend | Angular (pendiente, se hará después del backend) |

---

## Requisitos previos

- JDK 25 (o 21 si tu entorno lo requiere)
- Docker Desktop
- Git
- Un editor con soporte Java (VS Code con *Extension Pack for Java*, o IntelliJ)

---

## Puesta en marcha

### 1. Configurar las variables de entorno

Crea un archivo `.env` en la raíz del proyecto con tus propios valores:

```
POSTGRES_DB=juegoteca
POSTGRES_USER=tu_usuario
POSTGRES_PASSWORD=tu_contraseña
```

El archivo `.env` **no debe subirse a Git**. Comprueba que está incluido en `.gitignore`.

### 2. Arrancar la base de datos

```
docker compose up -d
```

Comprueba que el contenedor está activo:

```
docker ps
```

Debe aparecer `juegoteca-db` con estado `Up`.

Otros comandos útiles:

| Comando | Qué hace |
|---|---|
| `docker compose logs db` | Muestra los registros de PostgreSQL |
| `docker compose down` | Para y elimina el contenedor, **conserva los datos** |
| `docker compose down -v` | Lo anterior y además **borra los datos** |

### 3. Arrancar la aplicación

Desde la raíz del proyecto:

```
.\mvnw spring-boot:run
```

También puedes ejecutar la clase `JuegotecaApplication` desde tu editor, siempre con la raíz del proyecto como directorio de trabajo para que Spring encuentre el `.env`.

Si todo va bien, verás en la consola `HikariPool-1 - Start completed` (conexión con la base de datos) y `Started JuegotecaApplication`. El servidor queda escuchando en `http://localhost:8080`.

---

## Configuración

`src/main/resources/application.properties`:

```properties
spring.application.name=juegoteca
spring.config.import=optional:file:.env[.properties]
spring.datasource.url=jdbc:postgresql://localhost:5432/${POSTGRES_DB}
spring.datasource.username=${POSTGRES_USER}
spring.datasource.password=${POSTGRES_PASSWORD}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.open-in-view=false
```

- Spring importa el `.env`, así las credenciales viven en un solo sitio y fuera del control de versiones.
- `ddl-auto=update` deja que Hibernate cree y actualice las tablas a partir de las entidades. Es cómodo durante el aprendizaje, pero **se sustituirá por migraciones con Flyway** cuando el modelo esté estable.
- `open-in-view=false` evita mantener la conexión a la base de datos abierta durante toda la petición web.

---

## Modelo de datos

Tres tablas. `game_entries` es la tabla intermedia de la relación muchos a muchos entre usuarios y juegos, y guarda los datos propios de cada usuario con cada juego.

```
users 1 ────< game_entries >──── 1 games
```

### `users`

| Columna | Tipo | Restricciones |
|---|---|---|
| id | bigint | PK, autogenerada |
| username | varchar | único, no nulo |
| email | varchar | único, no nulo |
| password_hash | varchar | no nulo (nunca se guarda la contraseña en claro) |
| created_at | timestamp | no nulo |

### `games`

| Columna | Tipo | Restricciones |
|---|---|---|
| id | bigint | PK, autogenerada |
| igdb_id | bigint | único, no nulo (id del juego en IGDB) |
| name | varchar | no nulo |
| slug | varchar | |
| released_at | date | |
| cover_image_id | varchar | |

### `game_entries`

| Columna | Tipo | Restricciones |
|---|---|---|
| id | bigint | PK, autogenerada |
| user_id | bigint | FK a `users`, no nulo |
| game_id | bigint | FK a `games`, no nulo |
| status | varchar | no nulo (`WANT_TO_PLAY`, `PLAYING`, `FINISHED`, `ABANDONED`) |
| rating | smallint | nulo o entre 1 y 10 |
| review | text | |
| played_at | date | |
| created_at | timestamp | no nulo |
| updated_at | timestamp | no nulo |

Restricciones adicionales:

- `UNIQUE (user_id, game_id)`: un usuario tiene como máximo una ficha por juego.
- `CHECK (rating BETWEEN 1 AND 10)`.

### Reglas de negocio (capa de servicio)

- Nota, reseña y fecha de juego solo se permiten si el estado es `PLAYING`, `FINISHED` o `ABANDONED`.
- Nota y reseña son opcionales e independientes entre sí.

### Decisiones de diseño

- La tabla se llama `users` y no `user`, porque `user` es una palabra reservada en PostgreSQL.
- El estado se guardará como texto (`EnumType.STRING`) y no como número, para que reordenar el enum no cambie el significado de los datos ya guardados.
- La nota es un entero, sin decimales, para evitar problemas de precisión.

---

## Estructura prevista del código

```
src/main/java/.../juegoteca
├── entity       Entidades JPA (User, Game, GameEntry, EntryStatus)
├── repository   Acceso a datos con Spring Data
├── service      Reglas de negocio
├── controller   Endpoints REST
└── dto          Objetos de entrada y salida de la API
```

---

## Hoja de ruta

- [x] **1. Entorno:** proyecto Spring Boot, PostgreSQL en Docker y conexión funcionando.
- [x] **2. Diseño del modelo de datos:** diagrama entidad-relación.
- [ ] **3. Entidades JPA:** `User`, `Game`, `GameEntry` y `EntryStatus`.
- [ ] **4. API REST básica:** controller, service y repository, DTOs, validaciones y manejo global de errores.
- [ ] **5. Integración con IGDB:** autenticación OAuth2 con Twitch, búsqueda e importación de juegos.
- [ ] **6. Seguridad:** registro, login y JWT con Spring Security.
- [ ] **7. Funciones del MVP:** estados, notas, reseñas y fecha de juego.
- [ ] **8. Migraciones con Flyway** en lugar de `ddl-auto=update`.
- [ ] **9. Tests:** unitarios de servicios y de integración de la API.
- [ ] **10. Frontend en Angular.**
- [ ] **11. Despliegue:** Dockerfile, `docker-compose` completo y variables de entorno.

---

## Notas

- Los datos de videojuegos proceden de [IGDB](https://www.igdb.com). Revisa sus condiciones de uso antes de publicar la aplicación.
- IGDB requiere credenciales de una aplicación de Twitch (`client_id` y `client_secret`). Se añadirán al `.env` cuando se integre la API y tampoco se subirán a Git.
