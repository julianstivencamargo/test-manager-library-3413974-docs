# Documento de Implementación: T-01 — Esqueleto de la Aplicación y Endpoint de Salud

## 1. Identificador y Objetivo de la Tarea
- **ID de la Tarea:** T-01 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Objetivo:** Inicializar el esqueleto del backend en PHP con Laravel dentro de `test-manager-library-3413974-api`, aplicando la estructura base para Arquitectura Hexagonal (puertos y adaptadores), desacoplando el dominio de Laravel y HTTP, configurando el contenedor Docker para ejecución autónoma y con dependencias instaladas en la construcción, y exponiendo el endpoint de salud funcional.

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

## 3. Decisiones Técnicas y Arquitectónicas
- **Entorno de Ejecución y Contenedorización Autónoma:**
  - El `Dockerfile` se diseña como multi-stage o con Composer integrado para instalar automáticamente las dependencias del proyecto (`composer install --no-dev --optimize-autoloader`) durante la construcción de la imagen (`docker build`).
  - La imagen generada es autónoma y ejecutable desde un clon limpio sin requerir un directorio `vendor/` previo en el sistema host.
- **Estructura de Capas Hexagonales y Autoloading PSR-4:**
  - El namespace configurado en `composer.json` es `"Library\\": "src/"` (mapeando `src/Domain`, `src/Application`, `src/Adapters`, `src/Bootstrap`).
  - Para asegurar la persistencia en Git de toda la estructura de carpetas de la arquitectura hexagonal, cada directorio base incorpora un archivo de anclaje (`.gitkeep` o interfaz base):
    - `src/Domain/.gitkeep`
    - `src/Application/Ports/Outbound/.gitkeep`
    - `src/Application/Services/.gitkeep`
    - `src/Adapters/Inbound/HTTP/Controllers/HealthCheckController.php`
    - `src/Adapters/Outbound/Persistence/.gitkeep`
    - `src/Adapters/Outbound/Security/.gitkeep`
    - `src/Adapters/Outbound/Storage/.gitkeep`
    - `src/Bootstrap/Composition/.gitkeep`
- **Rutas de Salud:** Se mantienen `GET /health` y `GET /api/v1/health` retornando `{"status":"ok"}`.

## 4. Archivos Previstos
- `test-manager-library-3413974-docs/spec-laravel/t01-skeleton-health.md` (este documento).
- `test-manager-library-3413974-api/Dockerfile`: Construcción con Composer y dependencias integradas en el contenedor.
- `test-manager-library-3413974-api/docker-compose.yml`: Ejecución mediante un solo comando.
- `test-manager-library-3413974-api/composer.json`: Con autoloading `"Library\\": "src/"` documentado fielmente.
- Archivos `.gitkeep` en cada paquete de `src/` para garantizar la presencia de la estructura hexagonal en el repositorio.
- `test-manager-library-3413974-api/tests/Feature/HealthCheckTest.php`: Prueba automatizada de feature.

## 5. Criterios de Aceptación y Evidencias
- **CA-T01-1:** La petición HTTP `GET /health` (y `GET /api/v1/health`) responde código 200 con JSON `{"status":"ok"}`.
- **CA-T01-2:** El contenedor se construye y arranca con un solo comando desde un entorno sin `vendor/` local, instalando sus dependencias en la imagen Docker.
- **CA-T01-3:** Las pruebas automatizadas pasan al 100%.
- **CA-T01-4:** La estructura completa de carpetas hexagonal se encuentra versionada en el árbol de Git.
