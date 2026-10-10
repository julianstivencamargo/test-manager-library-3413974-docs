# T-20 a T-23 — Especificación del Ecosistema: Infraestructura, Utilidades, Frontend y Sitio Público

## 1. Identificadores y Alcance
- **Tareas Cubiertas:**
  - **T-20:** Infraestructura global con Docker Compose (`test-manager-library-3413974-infra`).
  - **T-21:** Sembrador de datos de demostración (`test-manager-library-3413974-tool`).
  - **T-22:** Frontend interactivo en React (`test-manager-library-3413974-app`).
  - **T-23:** Sitio público estático de presentación (`test-manager-library-3413974-page`).
- **Propósito:** Brindar la solución completa integrada y lista para la feria de presentación.

---

## 2. Infraestructura Global (T-20 — `infra`)

### 2.1. Arquitectura de Contenedores
El archivo `docker-compose.yml` orquesta los 4 servicios requeridos:
1. `db`: Motor PostgreSQL 16 Alpine en puerto `5432`. Arranca con la base de datos limpia `manager_library` y esquema `library`.
2. `api`: Backend Laravel 11 en puerto `8000`, configurado con variables de entorno para PostgreSQL y credenciales de administrador inicial.
3. `app`: Aplicación React en puerto `3000` servida mediante Nginx, configurada para comunicarse con `http://localhost:8000/api/v1`.
4. `page`: Sitio estático de presentación en puerto `8080` servido mediante Nginx alpine.

### 2.2. Red y Volúmenes
- Red compartida tipo puente: `manager-library-net`.
- Volumen persistente para la base de datos: `db_data`.

---

## 3. Herramienta de Demostración (T-21 — `tool`)

### 3.1. Propósito y Criterios
Aprovisionar mediante llamadas HTTP a la API o mediante comando de consola un conjunto de datos realista para la feria:
- **$\ge$ 40 libros:** 42 libros reales divididos equitativamente en las 5 categorías oficiales (`Novela`, `Ciencia`, `Tecnología`, `Historia`, `Infantil`), con ISBN-13 válidos y existencias controladas.
- **10 socios:** Con nombres y documentos de identidad colombianos válidos.
- **15 préstamos:** Cubriendo préstamos abiertos, cerrados (devueltos) y exactamente **3 préstamos con líneas vencidas** para validar inmediatamente:
  - `GET /api/v1/loans/overdue` (muestra líneas vencidas con cálculo de días de mora).
  - `GET /api/v1/reports/most-borrowed` (muestra filas consolidadas por libro y categoría congelada).

### 3.2. Formas de Ejecución
- **Vía comando CLI en API:** `docker compose run --rm test php artisan library:demo-seed`
- **Vía script HTTP:** `API_URL=http://localhost:8000/api/v1 php seed.php`

---

## 4. Frontend de Gestión en React (T-22 — `app`)

### 4.1. Características de la Aplicación (SPA)
1. **Autenticación:** Formulario de acceso que almacena el token JWT y el rol del usuario, con opción de cierre de sesión.
2. **Catálogo de Libros:** Búsqueda en vivo por título y autor, filtro por categoría, vista de stock en tiempo real, modal de creación de libros (rol admin) y baja lógica.
3. **Módulo de Socios:** Registro de nuevos socios con validación de documento único y consulta de socios con conteo de préstamos en mora.
4. **Módulo de Préstamos:** Selección de socio, selector interactivo de 1 a 3 libros disponibles, creación atómica de préstamo y botón de devolución línea por línea.
5. **Módulo de Vencidos:** Tabla operativa de libros en mora con badges de días de retraso.
6. **Reporte de Más Prestados:** Selector de rango de fechas con visualización del ranking de libros prestados sobre las categorías históricamente congeladas.

---

## 5. Sitio Público Estático (T-23 — `page`)

### 5.1. Características
- Sitio HTML5/CSS3 moderno y responsive sin dependencias de backend ni base de datos.
- Presenta el propósito del sistema, historias de usuario, invariantes de negocio (RN-1 a RN-6), arquitectura hexagonal y tecnologías del reto.
- Accesible en cualquier navegador web abriendo `index.html` o sirviéndolo en el puerto `8080`.
