# Documento de Implementación: T-01 — Esqueleto de la Aplicación y Endpoint de Salud

## 1. Identificador y Objetivo de la Tarea
- **ID de la Tarea:** T-01 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Objetivo:** Inicializar el esqueleto del backend en PHP con Laravel dentro de `test-manager-library-3413974-api`, aplicando la estructura base para Arquitectura Hexagonal (puertos y adaptadores), desacoplando el dominio de Laravel y HTTP, configurando el contenedor Docker para ejecución con un solo comando y exponiendo el endpoint de salud funcional.

## 2. Requisitos de la Especificación Aplicables
- **`tasks.md` T-01:** «Esqueleto del proyecto y `GET /health` | Evidencia: `GET /health` responde `{"status":"ok"}`; el contenedor arranca con un comando».
- **`constitution.md` Artículos I a IV:**
  - **Art. I:** El dominio no depende de nada (cero dependencias de Laravel, HTTP, ORM).
  - **Art. II:** El núcleo declara sus puertos; el mundo los implementa.
  - **Art. III:** Un único punto de composición (Service Providers / Bootstrap bindings).
  - **Art. IV:** Principios SOLID (SRP, OCP, LSP, ISP, DIP).
- **`api-contract.md` §1 y §3.1:**
  - Convención base: prefijo `/api/v1` para la API, pero compatibilidad explícita con `/health` y `/api/v1/health` para cumplir tanto con la convención base como con la evidencia estricta de `tasks.md`.
  - Endpoint de salud: público, responde `{"status": "ok"}` con código HTTP 200.

## 3. Decisiones Técnicas y Arquitectónicas
- **Entorno de Ejecución:** Contenedores Docker (sin instalaciones en el host de Windows). Se crea un `Dockerfile` en `test-manager-library-3413974-api` basado en PHP 8.3 CLI/FPM con Composer y extensiones básicas, y configuración de ejecución rápida (`php artisan serve` en puerto 8000).
- **Estructura de Capas Hexagonales dentro de `app/` (o `src/`):**
  - `src/Domain/`: Modelos, agregados y objetos de valor independientes (PHP estándar puro).
  - `src/Application/`: Casos de uso (servicios) y puertos (`Application/Ports/Outbound/`).
  - `src/Adapters/Inbound/HTTP/`: Controladores, Requests, Middlewares y transformadores.
  - `src/Adapters/Outbound/`: Implementaciones de persistencia, seguridad y storage.
  - `src/Bootstrap/`: Proveedores de enlace e inyección de dependencias (Service Provider de composición).
- **Rutas de Salud:** Se habilitan tanto `GET /health` como `GET /api/v1/health` retornando `{"status":"ok"}` en formato JSON, asegurando total conformidad con la evidencia de `tasks.md` y la convención de `api-contract.md`.

## 4. Archivos Previstos
- `test-manager-library-3413974-docs/spec-laravel/t01-skeleton-health.md` (este documento).
- `test-manager-library-3413974-api/Dockerfile`: Definición de contenedor para el backend Laravel.
- `test-manager-library-3413974-api/docker-compose.yml` (o script de arranque standalone para T-01): Permite arrancar la API con un solo comando (`docker compose up` o `docker run`).
- `test-manager-library-3413974-api/composer.json`: Definición del proyecto Laravel 11/12 con autoloading configurado para las capas de arquitectura hexagonal (`App\\Domain\\`, `App\\Application\\`, `App\\Adapters\\`).
- `test-manager-library-3413974-api/bootstrap/app.php`: Punto de entrada de la aplicación Laravel.
- `test-manager-library-3413974-api/routes/api.php` y `routes/web.php`: Registro de la ruta de salud.
- `test-manager-library-3413974-api/src/Adapters/Inbound/HTTP/Controllers/HealthCheckController.php`: Controlador HTTP de salud.
- `test-manager-library-3413974-api/tests/Feature/HealthCheckTest.php`: Prueba automatizada de feature que comprueba la respuesta 200 y el JSON `{"status":"ok"}`.

## 5. Criterios de Aceptación y Evidencias
- **CA-T01-1:** La petición HTTP `GET /health` (y `GET /api/v1/health`) responde código 200 con JSON `{"status":"ok"}`.
- **CA-T01-2:** El contenedor de la API arranca con un solo comando sin errores.
- **CA-T01-3:** Las pruebas automatizadas (PHPUnit/Pest en contenedor) se ejecutan y pasan al 100%.
- **CA-T01-4:** Estructura de carpetas hexagonal inicializada y documentada.
