# Documento de Implementación: T-02 — Migración Inicial de la Base de Datos

## 1. Identificador y Objetivo de la Tarea
- **ID de la Tarea:** T-02 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Objetivo:** Implementar la migración inicial de la base de datos en Laravel (`test-manager-library-3413974-api`), creando el esquema físico para las seis entidades del dominio en PostgreSQL 16 (`category`, `book`, `member`, `user`, `loan`, `loan_item`), definiendo sus claves primarias UUID, claves foráneas, restricciones de integridad, índices únicos/parciales requeridos y sembrando las cinco categorías fijas con sus identificadores inmutables, respetando la regla innegociable de cero columnas con valor `DEFAULT`.

## 2. Requisitos de la Especificación Aplicables
- **`tasks.md` T-02:** «Migración inicial: seis tablas, FK, índices únicos y las cinco categorías sembradas | Evidencia: Las consultas de `data-model.md` §10 devuelven lo documentado; ninguna columna con `DEFAULT`».
- **`constitution.md`:**
  - **Art. V:** El esquema es propiedad de las migraciones de la aplicación (ADR-001). `infra` solo levanta el motor vacío; todo DDL vive en la migración.
  - **Art. VI:** Las invariantes que la base puede imponer, la base las impone (PK, FK, UNIQUE, CHECK).
  - **Art. VII:** Lo derivado se calcula, no se almacena (no existen columnas `status` ni `days_late`).
  - **Art. VIII:** Los valores congelados no siguen al catálogo (título, ISBN y categoría congelados en `loan_item`).
  - **Art. XI:** Nombres de tablas en singular y snake_case inglés (`category`, `book`, `member`, `user`, `loan`, `loan_item`).
- **`data-model.md`:**
  - §0, §3 y §4: Definición exacta de columnas, tipos (`uuid`, `varchar`, `char(13)`, `integer`, `timestamptz`), nulabilidad y restricciones.
  - Ninguna columna lleva cláusula `DEFAULT`.
  - Esquema `library` en PostgreSQL.
  - Restricción de verificación `ck_book_available_range`: `0 <= available_copies <= total_copies`.
  - Índice único parcial en `book`: `isbn` único donde `deleted_at IS NULL`.
  - Claves foráneas:
    - `book.category_id` $\rightarrow$ `category.id` (RESTRICT)
    - `loan.member_id` $\rightarrow$ `member.id` (RESTRICT)
    - `loan.loaned_by_user_id` $\rightarrow$ `user.id` (RESTRICT)
    - `loan_item.loan_id` $\rightarrow$ `loan.id` (ON DELETE CASCADE)
    - `loan_item.book_id` $\rightarrow$ `book.id` (RESTRICT)
  - §9: Las 5 categorías fijas sembradas en la migración inicial:
    - `11111111-1111-4111-8111-111111111111`: Novela
    - `22222222-2222-4222-8222-222222222222`: Ciencia
    - `33333333-3333-4333-8333-333333333333`: Tecnología
    - `44444444-4444-4444-8444-444444444444`: Historia
    - `55555555-5555-4555-8555-555555555555`: Infantil

## 3. Decisiones Técnicas y Arquitectónicas (Docker-First)
- **Motor de Base de Datos y Servicio Docker:**
  - Se añade el servicio `db` basado en la imagen oficial `postgres:16-alpine` al `docker-compose.yml` de la API para soportar la ejecución y prueba aislada de las migraciones sin tocar Windows ni instalar PostgreSQL localmente.
  - Se habilita la extensión `pdo_pgsql` en el contenedor de la API (añadiendo `postgresql-dev` y ejecutando `docker-php-ext-install pdo_pgsql` en el `Dockerfile`).
- **Limpieza de migraciones por defecto de Laravel:**
  - Las migraciones predeterminadas de Laravel (`0001_01_01_000000_create_users_table.php`, `...cache...`, `...jobs...`) se retiran o sustituyen para no introducir tablas ni columnas ajenas al modelo de datos formal (como `remember_token`, `email`, `created_at`, `updated_at`).
- **Estructura DDL de la Migración:**
  - Creación del esquema `library` si no existe (`CREATE SCHEMA IF NOT EXISTS library;`).
  - Creación de las seis tablas con tipos estrictos PostgreSQL (`uuid`, `timestamptz`).
  - Declaración explícita de `ck_book_available_range`.
  - Declaración del índice único condicional `CREATE UNIQUE INDEX uq_book_isbn_active ON library.book (isbn) WHERE deleted_at IS NULL;`.
  - Inserción directa de las 5 categorías fijas dentro de la transacción de la migración.
  - Cero directivas `->default()` en las definiciones de columnas de Blueprint / SQL.

## 4. Archivos Previstos
- `test-manager-library-3413974-docs/spec-laravel/t02-initial-migration.md` (este documento).
- `test-manager-library-3413974-api/Dockerfile`: Inclusión de soporte para extensión PHP `pdo_pgsql`.
- `test-manager-library-3413974-api/docker-compose.yml`: Inclusión del servicio PostgreSQL 16 (`db`) y variables de entorno correspondientes.
- `test-manager-library-3413974-api/database/migrations/2026_10_10_000001_create_library_schema_and_tables.php`: Migración inicial canónica con las seis tablas, restricciones, índices y semilla.
- `test-manager-library-3413974-api/tests/Feature/InitialMigrationTest.php`: Suite automatizada que ejecuta las consultas de verificación de `data-model.md` §10.

## 5. Criterios de Aceptación y Verificación
- **CA-T02-1:** La migración se ejecuta con éxito mediante `docker compose run --rm api php artisan migrate`.
- **CA-T02-2:** Las consultas de `data-model.md` §10 sobre `information_schema.columns` confirman que ninguna columna tiene `column_default` (cero defaults).
- **CA-T02-3:** Las seis tablas existen dentro del esquema `library`: `category`, `book`, `member`, `user`, `loan`, `loan_item`.
- **CA-T02-4:** Las restricciones `ck_book_available_range`, FKs y los índices únicos (incluyendo el parcial para `isbn`) existen y son activos en PostgreSQL.
- **CA-T02-5:** Las cinco categorías fijas están presentes con sus UUIDs exactos.
- **CA-T02-6:** Las pruebas automatizadas en `InitialMigrationTest.php` pasan al 100% en Docker.
