# Documento de Implementación: T-01 — Esqueleto de la Aplicación y Endpoint de Salud

## 1. Identificador y Objetivo de la Tarea
- **ID de la Tarea:** T-01 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Objetivo:** Inicializar el esqueleto del backend en PHP con Laravel dentro de `test-manager-library-3413974-api`, aplicando la estructura base para Arquitectura Hexagonal (puertos y adaptadores), desacoplando el dominio de Laravel y HTTP, configurando el contenedor Docker para ejecución autónoma y reproducible, y exponiendo el endpoint de salud funcional.

## 2. Requisitos de la Especificación Aplicables
- **`tasks.md` T-01:** «Esqueleto del proyecto y `GET /health` | Evidencia: `GET /health` responde `{"status":"ok"}`; el contenedor arranca con un comando».
- **`constitution.md` Artículos I a IV:**
  - **Art. I:** El dominio no depende de nada (cero dependencias de Laravel, HTTP, ORM).
  - **Art. II:** El núcleo declara sus puertos; el mundo los implementa.
  - **Art. III:** Un único punto de composición (Service Providers / Bootstrap bindings).
  - **Art. IV:** Principios SOLID (SRP, OCP, LSP, ISP, DIP).
- **`api-contract.md` §1 y §3.1:**
  - Convención base: prefijo `/api/v1` para la API, con soporte tanto en `/health` como en `/api/v1/health`.
  - Endpoint de salud: público, responde `{"status": "ok"}` con código HTTP 200.

## 3. Decisiones Técnicas y Arquitectónicas (Docker-First y Aislamiento)
- **Principio Docker-First estricto:** Cero instalaciones o dependencias en el host Windows. Toda instalación, compilación, ejecución y prueba corre exclusivamente en contenedores.
- **Construcción y Dependencias (Dev + Runtime):**
  - El `Dockerfile` instala tanto dependencias de ejecución como de desarrollo (`composer install --prefer-dist --optimize-autoloader`) para que herramientas de pruebas como PHPUnit estén disponibles dentro de la imagen.
  - Se incluye un script/servicio para ejecutar la suite de pruebas PHPUnit (`docker compose run --rm test` o `docker compose run --rm api php artisan test`) de forma 100% reproducible y aislada.
- **Preservación de `vendor` en `docker-compose.yml`:**
  - Al montar volúmenes locales (`.:/app`) para desarrollo activo, se define un volumen anónimo dedicado para `/app/vendor`. Esto evita que un directorio vacío del host oculte el directorio `/app/vendor` generado durante la construcción de la imagen.
- **Estructura Hexagonal y Autoloading:**
  - Namespace `"Library\\": "src/"` en `composer.json`.
  - Capas hexagonales explícitamente versionadas en Git (`Domain`, `Application`, `Adapters`, `Bootstrap`).

## 4. Archivos Previstos
- `test-manager-library-3413974-docs/spec-laravel/t01-skeleton-health.md` (este documento).
- `test-manager-library-3413974-api/Dockerfile`: Construcción con dependencias completas y autoload optimizado.
- `test-manager-library-3413974-api/docker-compose.yml`: Montaje con volumen anónimo para `/app/vendor` y servicio `test` reproducible.
- `test-manager-library-3413974-api/tests/Feature/HealthCheckTest.php`: Prueba automatizada de feature.

## 5. Criterios de Aceptación y Evidencias
- **CA-T01-1:** La petición HTTP `GET /health` (y `GET /api/v1/health`) responde código 200 con JSON `{"status":"ok"}`.
- **CA-T01-2:** En un clon limpio sin `vendor/` local, `docker compose up` levanta y responde sin error de dependencias faltantes.
- **CA-T01-3:** Las pruebas de PHPUnit se ejecutan dentro del contenedor (`docker compose run --rm api php artisan test` o similar) y pasan al 100%.
- **CA-T01-4:** Estructura de carpetas hexagonal versionada y documentada.
