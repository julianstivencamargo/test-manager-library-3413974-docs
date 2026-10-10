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
