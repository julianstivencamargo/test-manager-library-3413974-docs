# T-07 a T-17 — Especificación Técnica del Núcleo de la API y Adaptadores

## 1. Identificadores y Alcance
- **Tareas Cubiertas:**
  - **T-07:** Autenticación y creación de bibliotecarios (`POST /auth/login`, `POST /users`).
  - **T-08:** Gestión y búsqueda de libros (`GET /books`, `POST /books`, `PUT /books/{id}`, `DELETE /books/{id}`, portadas).
  - **T-09:** Registro atómico de préstamos (`POST /loans`).
  - **T-10:** Devolución de ejemplares y consulta de vencidos (`POST /loans/{loan_id}/items/{item_id}/return`, `GET /loans/overdue`).
  - **T-11:** Reporte de libros más prestados (`GET /reports/most-borrowed`).
  - **T-12 a T-15:** Adaptadores de persistencia (Eloquent con esquema `library`), seguridad (JWT HS256, hash Bcrypt) y almacenamiento de portadas.
  - **T-16:** Adaptador HTTP completo con cuerpo de error único y autorización por roles.
  - **T-17:** Aprovisionamiento automático del administrador desde variables de entorno.
- **Prefijo de Rutas:** `/api/v1`

---

## 2. Seguridad y Tokens JWT (RFC 7519)

### 2.1. Estructura del Token
- **Algoritmo:** `HS256` firmado con la clave de aplicación (`APP_KEY`).
- **Expiración:** 28 800 segundos (8 horas continuas).
- **Claims incluidos:**
  - `sub`: Identificador UUID del usuario autenticado.
  - `role`: Rol del usuario (`admin` o `librarian`).
  - `iat`: Instante de emisión en formato timestamp Unix.
  - `exp`: Instante de caducidad ($iat + 28800$).
- **Privacidad (RNF-3):** El token no contiene contraseñas, correos ni datos personales.

### 2.2. Middleware de Autenticación y Autorización
- `AuthenticateJwt`: Inspecciona la cabecera `Authorization: Bearer <token>`. Si falta o es inválido, responde 401 con código `INVALID_CREDENTIALS`.
- `RequireRole`: Valida que el rol del usuario corresponda con los permisos del endpoint. Si no coincide, responde 403 con código `FORBIDDEN_ROLE`.

---

## 3. Contrato de Respuestas y Manejo Uniforme de Errores

De acuerdo con `api-contract.md`, toda respuesta de error HTTP (4xx y 5xx) mantiene estrictamente la misma estructura:
```json
{
  "code": "CODIGO_DE_NEGOCIO",
  "message": "Mensaje descriptivo en español.",
  "details": [
    { "field": "campo_afectado", "issue": "descripción del problema" }
  ]
}
```

### Matriz de Códigos de Error
| Código de Negocio | HTTP Status | Causa |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | Credenciales inválidas o token ausente/expirado |
| `FORBIDDEN_ROLE` | 403 | Rol del usuario insuficiente para la operación |
| `NOT_FOUND` | 404 | Recurso inexistente o dado de baja |
| `USERNAME_TAKEN` | 409 | Nombre de usuario duplicado en creación de bibliotecario |
| `ISBN_TAKEN` | 409 | ISBN duplicado entre libros activos |
| `DOCUMENT_TAKEN` | 409 | Documento duplicado en creación de socio |
| `BOOK_HAS_OPEN_LOANS` | 409 | Intento de baja lógica a libro con préstamos activos |
| `BOOK_NOT_AVAILABLE` | 409 | Intento de préstamo de libro sin ejemplares disponibles |
| `MEMBER_HAS_OVERDUE` | 409 | Socio con préstamos vencidos intenta nuevo préstamo |
| `ITEM_ALREADY_RETURNED` | 409 | Intento de devolver una línea ya devuelta |
| `MEMBER_LIMIT_EXCEEDED` | 422 | Socio intenta exceder el límite de 3 libros sin devolver |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | Mismo libro repetido en una solicitud de préstamo |
| `VALIDATION_ERROR` | 422 | Error en datos de entrada o rol admin solicitado vía API |
| `INVALID_DATE_RANGE` | 422 | Rango de fechas invertido ($to < from$) |

---

## 4. Endpoints y Casos de Uso Implementados

### 4.1. Autenticación y Salud
- `GET /health` y `GET /api/v1/health`: Devuelve `{"status":"ok"}`.
- `POST /api/v1/auth/login`: Autentica usuario y clave; retorna token Bearer con vigencia de 8 horas y rol.

### 4.2. Usuarios (Admin)
- `POST /api/v1/users`: Crea un bibliotecario (`role: librarian`). Si se pide rol `admin` responde 422. Retorna usuario sin clave.

### 4.3. Categorías (Autenticado)
- `GET /api/v1/categories`: Lista las 5 categorías inmutables ordenadas alfabéticamente.

### 4.4. Catálogo de Libros (Admin y Bibliotecario)
- `GET /api/v1/books`: Búsqueda predictiva por texto (`q`), categoría (`category_id`), paginación (`page`, `size`). Solo libros activos.
- `GET /api/v1/books/{id}`: Detalle completo de libro.
- `POST /api/v1/books` (Admin): Crea libro con título, autor, ISBN-13 validado, categoría y total de ejemplares.
- `PUT /api/v1/books/{id}` (Admin): Edita libro y ajusta copias disponibles según la diferencia.
- `DELETE /api/v1/books/{id}` (Admin): Baja lógica protegida si hay préstamos abiertos.
- `PUT /api/v1/books/{id}/cover` y `DELETE /api/v1/books/{id}/cover`: Carga y eliminación de portadas JPG/PNG (máximo 2 MB).
- `GET /api/v1/books/{id}/cover`: Descarga del binario de portada.

### 4.5. Socios (Admin y Bibliotecario)
- `GET /api/v1/members`: Búsqueda y listado paginado.
- `GET /api/v1/members/{id}`: Detalle de socio con conteo de préstamos activos sin devolver.
- `POST /api/v1/members`: Creación de socio con documento único.

### 4.6. Préstamos y Devoluciones (Admin y Bibliotecario)
- `POST /api/v1/loans`: Operación atómica transaccional de 1 a 3 libros. Descuenta stock, verifica moras y límites de socio, y congela datos de línea.
- `GET /api/v1/loans`: Listado paginado con filtros y cálculo de estado dinámico (`open`, `overdue`, `closed`).
- `GET /api/v1/loans/{id}`: Detalle con líneas y estado calculado.
- `POST /api/v1/loans/{loan_id}/items/{item_id}/return`: Devolución de línea con incremento de disponibles.
- `GET /api/v1/loans/overdue`: Líneas vencidas no devueltas con cálculo de días de mora.

### 4.7. Reportes (Admin)
- `GET /api/v1/reports/most-borrowed`: Ranking consolidado de libros más prestados por rango de fechas sobre datos congelados.

---

## 5. Arranque y Composición
- Seeder `AdminUserSeeder` y comando `php artisan library:bootstrap-admin` leen `ADMIN_USERNAME` y `ADMIN_PASSWORD` desde variables de entorno y crean el administrador inicial si no existe.
- Suite de verificación: `tests/Feature/ApiContractTest.php` (7 pruebas de contrato completas, 57 aserciones).

### 5.1. Configuración de Entorno de Persistencia (PostgreSQL)
- Para la correcta resolución en el entorno de desarrollo y orquestación Docker, la configuración en las variables de entorno del backend debe establecer inequívocamente `DB_CONNECTION=pgsql`, `DB_HOST=db`, `DB_PORT=5432`, `DB_DATABASE=manager_library`, `DB_USERNAME=library_user`, `DB_PASSWORD=library_secret` y `DB_SCHEMA=library`.
- Esto garantiza que el runtime de Laravel utilice el driver `pgsql` y el search path `library` de PostgreSQL 16 provisto por el contenedor `manager-library-db`, evitando fallbacks no deseados al driver SQLite.

### 5.2. Política Estricta de Seguridad para Archivos `.env` (RNF-Seguridad)
- **Exclusión Absoluta del Control de Versiones:** Los archivos con valores de entorno reales o locales (`.env`, `.env.local`, `.env.*`) no deben evidenciarse ni versionarse bajo ninguna circunstancia en ninguno de los repositorios del proyecto.
- **Configuración de `.gitignore`:** Todo repositorio del ecosistema debe mantener reglas explícitas en su `.gitignore` para ignorar `.env` y variantes locales.
- **Plantillas Referenciales:** Únicamente se permite versionar `.env.example` en repositorios que lo requieran como plantilla ilustrativa sin secretos sensibles de producción (`APP_KEY` vacía), sirviendo exclusivamente como guía de despliegue.
- **Exclusión de la imagen Docker:** El `.dockerignore` del backend excluye `.env` y sus variantes, de modo que ningún archivo de entorno local se copie dentro de la imagen construida.

### 5.3. Arranque desde un clon limpio (T-01, T-17, T-20)
Un clon recién descargado no contiene `.env`. El contenedor del backend usa un *entrypoint* (`docker/entrypoint.sh`) que, antes de ejecutar el comando principal:
1. Si no existe `.env`, lo crea copiando `.env.example` (el archivo resultante queda ignorado por Git).
2. Si `APP_KEY` está vacía, genera una clave aleatoria local con `php artisan key:generate`. La clave nunca se versiona (constitución, «Todo secreto viene del entorno»).
3. Limpia la caché de configuración.

El entrypoint **no ejecuta migraciones**: la base arranca sin tablas hasta que se corre `php artisan migrate` (evidencia de T-20).

**Administrador inicial automático (T-17):** al terminar `php artisan migrate`, un listener del evento `MigrationsEnded`, registrado en el punto de composición (`AppServiceProvider`), ejecuta `AdminUserSeeder`. El seeder crea el administrador con `ADMIN_USERNAME` y `ADMIN_PASSWORD` del entorno si aún no existe. Si `ADMIN_PASSWORD` no está definida, no crea el usuario y emite una advertencia: no hay contraseña por defecto en el código. El comando `php artisan library:bootstrap-admin` sigue disponible para ejecutarlo manualmente.

**Criterios de aceptación verificables:**
- **CA-ARR-1:** En un clon limpio, tras `docker compose up -d`, `GET /health` responde `200 {"status":"ok"}`.
- **CA-ARR-2:** Antes de migrar, el esquema `library` no contiene tablas.
- **CA-ARR-3:** Tras `php artisan migrate`, sin ejecutar ningún sembrador de demostración, `POST /api/v1/auth/login` con las credenciales del entorno responde `200` con `access_token`.

### 5.4. Aislamiento de las pruebas
Las pruebas de contrato (`ApiContractTest`) usan el trait `DatabaseTransactions`: cada prueba se ejecuta dentro de una transacción que se revierte al terminar, y no deja registros en la base de datos de demostración. Comando de ejecución con el ecosistema levantado: `docker compose exec api php artisan test` (desde `infra`).


