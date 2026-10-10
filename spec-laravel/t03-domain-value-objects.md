# Documento de Implementación: T-03 — Objetos de Valor del Dominio

## 1. Identificador y Objetivo de la Tarea
- **ID de la Tarea:** T-03 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Objetivo:** Implementar los objetos de valor fundamentales del dominio en PHP (`Isbn`, `Quantity`, `DateRange`), garantizando que sean tipos inmutables, autónomos y libres de dependencias de Laravel, HTTP, ORM o librerías de terceros (Artículo I de `constitution.md`), respaldados por pruebas unitarias completas.

## 2. Requisitos de la Especificación Aplicables
- **`tasks.md` T-03:** «Dominio: `Isbn`, `Quantity`, `DateRange` (objetos de valor) | Evidencia: Pruebas unitarias: ISBN de 13 dígitos, cantidad > 0, fin ≥ inicio».
- **`constitution.md`:**
  - **Art. I:** El dominio no depende de nada (solo PHP estándar: tipos escalares, `DateTimeImmutable`, excepciones nativas de dominio).
  - **Art. IV:** Principio de Responsabilidad Única (SRP).
  - **Art. XI:** Código, clases y nombres en inglés. Excepciones con mensajes en español.
- **`spec.md`:**
  - **CA-03.2:** El ISBN tiene 13 dígitos numéricos. Inválido si contiene letras, longitud diferente de 13 o caracteres no numéricos.
  - **CA-03.3 / RN-1:** Cantidades: Total de copias $\ge 1$. Las copias y transacciones numéricas no admiten cantidades menores a 1 para adición/creación.
  - **CA-10.2:** Rango de fechas: `end` no puede ser anterior a `start` (fin $\ge$ inicio). Ambas fechas en UTC.

## 3. Decisiones Técnicas y Arquitectónicas
- **Ubicación en el Hexágono:**
  - Namespace: `Library\Domain\ValueObjects\` en `test-manager-library-3413974-api/src/Domain/ValueObjects/`.
- **Inmutabilidad:**
  - Todas las clases se declaran `final readonly class`.
- **Excepciones de Dominio:**
  - `InvalidIsbnException`, `InvalidQuantityException`, `InvalidDateRangeException` en `Library\Domain\Exceptions\`. Heredan de `DomainException` o `InvalidArgumentException` nativas de PHP.
- **Especificaciones de cada Objeto de Valor:**
  1. `Isbn`:
     - Normalización: Recorta espacios en blanco y guiones si existen, asegurando exactamente 13 dígitos (`/^\d{13}$/`).
     - Invariante: Rechaza cadenas que no tengan exactamente 13 dígitos numéricos.
     - Métodos: `value(): string`, `equals(Isbn $other): bool`, `__toString(): string`.
  2. `Quantity`:
     - Invariante: Entero estrictamente positivo ($value > 0$).
     - Métodos: `value(): int`, `add(Quantity $other): Quantity`, `equals(Quantity $other): bool`.
  3. `DateRange`:
     - Invariante: `startDate <= endDate` (`endDate` no puede ser estrictamente anterior a `startDate`). Fechas expresadas en `DateTimeImmutable`.
     - Métodos: `startDate(): DateTimeImmutable`, `endDate(): DateTimeImmutable`, `includes(DateTimeImmutable $date): bool`.

## 4. Archivos Previstos
- `test-manager-library-3413974-docs/spec-laravel/t03-domain-value-objects.md` (este documento).
- `test-manager-library-3413974-api/src/Domain/ValueObjects/Isbn.php`
- `test-manager-library-3413974-api/src/Domain/ValueObjects/Quantity.php`
- `test-manager-library-3413974-api/src/Domain/ValueObjects/DateRange.php`
- `test-manager-library-3413974-api/src/Domain/Exceptions/InvalidIsbnException.php`
- `test-manager-library-3413974-api/src/Domain/Exceptions/InvalidQuantityException.php`
- `test-manager-library-3413974-api/src/Domain/Exceptions/InvalidDateRangeException.php`
- `test-manager-library-3413974-api/tests/Unit/Domain/ValueObjectsTest.php`: Suite unitaria pura sin base de datos ni HTTP.

## 5. Criterios de Aceptación y Evidencias
- **CA-T03-1:** `Isbn` válido con 13 dígitos se instancia correctamente; rechaza menos/más dígitos o caracteres no numéricos con `InvalidIsbnException`.
- **CA-T03-2:** `Quantity` con valor $> 0$ se instancia y permite operaciones aritméticas; rechaza $\le 0$ con `InvalidQuantityException`.
- **CA-T03-3:** `DateRange` con fin $\ge$ inicio es válido; fin anterior a inicio lanza `InvalidDateRangeException`.
- **CA-T03-4:** Cero dependencias externas en `src/Domain/` (solo PHP estándar).
- **CA-T03-5:** Las pruebas unitarias corren en contenedor Docker (`docker compose run --rm test`) y pasan al 100%.
