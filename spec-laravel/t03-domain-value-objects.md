# T-03 — Especificación Técnica y Criterios de Aceptación de los Objetos de Valor

## 1. Identificador y Alcance de la Tarea
- **ID de la Tarea:** T-03 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Propósito:** Documentar con exhaustividad técnica, contratos formales, invariantes, excepciones y pruebas unitarias los tres objetos de valor fundamentales del dominio:
  - `Isbn`
  - `Quantity`
  - `DateRange`
- **Ámbito:** Exclusivamente el núcleo de dominio puro (`src/Domain/ValueObjects` y `src/Domain/Exceptions`) en `test-manager-library-3413974-api`. Sin dependencias de frameworks ni infraestructura.

---

## 2. Reglas Generales de los Objetos de Valor

### 2.1. Inmutabilidad y Encapsulamiento
- Implementación como clases `final readonly class` en PHP 8.2+.
  - `final`: Prohíbe la herencia o extensión de la clase.
  - `readonly`: Garantiza que todas las propiedades se asignen exactamente una vez durante la instanciación y no puedan mutar.
- Validación inmediata en la construcción: Ninguna instancia puede existir en un estado inconsistente o inválido. Si los argumentos no cumplen las invariantes, se interrumpe la instanciación mediante la excepción de dominio correspondiente.
- Todas las operaciones que generen nuevos valores retornan una nueva instancia sin alterar el objeto receptor.
- Nombres de clases y métodos en inglés según el Artículo XI de `constitution.md`. Mensajes de excepción en español.

### 2.2. Independencia Absoluta del Framework (Artículo I)
- Ubicación estricta: `src/Domain/ValueObjects/` y `src/Domain/Exceptions/` bajo el namespace `Library\Domain\`.
- Cero dependencias de Laravel (sin usar `Illuminate\*`), Eloquent, bases de datos, HTTP Requests/Responses o paquetes de terceros.
- Solo funciones y tipos estándar de PHP (`string`, `int`, `bool`, `DateTimeImmutable`, `DateTimeZone`, `DomainException`, `InvalidArgumentException`).

### 2.3. Jerarquía y Contrato de Excepciones
Las excepciones de dominio se ubican en `Library\Domain\Exceptions\` y heredan de `DomainException` (o `InvalidArgumentException` de SPL nativa de PHP):
1. `InvalidIsbnException`: Representa una violación al contrato de formato o caracteres del ISBN.
2. `InvalidQuantityException`: Representa una violación a la regla de cantidad estrictamente positiva.
3. `InvalidDateRangeException`: Representa una violación a la relación temporal fin $\ge$ inicio.

---

## 3. Objeto de Valor `Isbn`

### 3.1. Propósito
Representar el identificador ISBN-13 de un libro del catálogo de manera inmutable y validada.

### 3.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\Isbn`
- **Constructor:** Constructor estricto `public function __construct(string $value)`
- **Tipo de entrada:** Exclusivamente `string`. El constructor declara tipado estricto `string $value` (con `declare(strict_types=1);`). No se admiten ni convierten silenciosamente enteros, flotantes, booleanos ni nulos (los cuales causan un `TypeError` a nivel de lenguaje).

### 3.3. Formato e Invariantes
- **Expresión de formato de referencia:** `^[0-9]{13}$` (exactamente 13 dígitos numéricos ASCII entre `0` y `9`).
- **Reglas de rechazo (lanzan `InvalidIsbnException`):**
  - Cadenas vacías o con espacios únicamente.
  - Cadenas con longitud menor a 13 dígitos (ej. 10 o 12 dígitos).
  - Cadenas con longitud mayor a 13 dígitos (ej. 14 dígitos).
  - Caracteres alfabéticos (ej. `"978123456789X"` o `"978012345678a"`).
  - Caracteres especiales o signos (ej. `"+9781234567890"`, `"-9781234567890"`).
  - Espacios interiores (ej. `"978 1234567890"`).
  - Notación científica o caracteres no numéricos.
- **Valores con ceros a la izquierda:** Se admiten siempre que cumplan con los 13 dígitos numéricos (ej. `"0123456789012"`).

### 3.4. Política de Normalización y Separadores
- **Política aprobada:** Se permite la eliminación de guiones (`-`) y espacios exteriores (`trim`) al momento de la entrada para facilitar la experiencia de usuario y compatibilidad con formatos estándar de edición, tras lo cual la cadena resultante debe coincidir exactamente con `^[0-9]{13}$`.
- **Procedimiento de normalización:**
  1. `trim($value)`: Remueve espacios al inicio y al final.
  2. `str_replace('-', '', $value)`: Remueve guiones separadores.
  3. Validación de la cadena resultante contra `/^[0-9]{13}$/`.
- El valor almacenado internamente y retornado por `value()` es la cadena limpia de 13 dígitos numéricos ASCII.
- Si tras remover guiones y espacios exteriores la cadena contiene caracteres no numéricos, espacios interiores o longitud distinta de 13, se lanza `InvalidIsbnException`.

### 3.5. Decisión Técnica: Validación de Formato vs. Dígito de Control
- **Contexto:** En `spec.md` (CA-03.2) y `data-model.md` (§3) se especifica textualmente: *«El ISBN tiene 13 dígitos y es único entre libros activos»* y *«`isbn char(13)`: Solo dígitos»*. En `tasks.md` T-03 se indica: *«Pruebas unitarias: ISBN de 13 dígitos»*.
- **Decisión cerrada (Alternativa A - Formato Estricto):** La especificación del reto exige la validación del formato estricto de 13 dígitos numéricos ASCII (`^[0-9]{13}$`). No se impone el algoritmo de suma ponderada mod 10 del ISBN-13 como requisito bloqueante, ya que los datos de prueba y ejemplos del spec (e.g. semillas sintéticas o catálogos simulados) definen el requisito como "13 dígitos numéricos".

### 3.6. Métodos Públicos
- `public function value(): string`: Retorna el ISBN normalizado de 13 dígitos.
- `public function equals(Isbn $other): bool`: Compara por igualdad de valor (`$this->value === $other->value()`).
- `public function __toString(): string`: Retorna el valor normalizado.

### 3.7. Pruebas Requeridas para `Isbn`
| # | Entrada | Resultado Esperado | Excepción |
|---|---|---|---|
| P-ISBN-01 | `"9789584276971"` | Instancia creada, `value() === "9789584276971"` | Ninguna |
| P-ISBN-02 | `"978-958-42-7697-1"` | Normalizado a `"9789584276971"` | Ninguna |
| P-ISBN-03 | `"  9789584276971  "` | Normalizado a `"9789584276971"` | Ninguna |
| P-ISBN-04 | `"0123456789012"` (comienza en 0) | Válido, preserva cero inicial | Ninguna |
| P-ISBN-05 | `""` (vacío) | Rechazado | `InvalidIsbnException` |
| P-ISBN-06 | `"978123456789"` (12 dígitos) | Rechazado por longitud insuficiente | `InvalidIsbnException` |
| P-ISBN-07 | `"97812345678901"` (14 dígitos) | Rechazado por exceso de longitud | `InvalidIsbnException` |
| P-ISBN-08 | `"978123456789X"` (con letra) | Rechazado por caracter alfabético | `InvalidIsbnException` |
| P-ISBN-09 | `"978-1234-5678-A"` | Rechazado por caracter alfabético | `InvalidIsbnException` |
| P-ISBN-10 | `"978 1234 56789"` (espacio interno) | Rechazado | `InvalidIsbnException` |
| P-ISBN-11 | `"+9781234567890"` (signo) | Rechazado | `InvalidIsbnException` |
| P-ISBN-12 | `1234567890123` (entero) | Rechazado por tipado estricto PHP | `TypeError` |
| P-ISBN-13 | Dos instancias con mismo valor | `equals()` retorna `true` | Ninguna |

---

## 4. Objeto de Valor `Quantity`

### 4.1. Propósito
Representar una cantidad de copias o ejemplares estrictamente positiva dentro del dominio (ej. total de ejemplares $\ge 1$).

### 4.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\Quantity`
- **Constructor:** `public function __construct(int $value)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente `int`. Se rechazan flotantes (`1.0`), cadenas numéricas (`"1"`), booleanos y nulos por tipado estricto.

### 4.3. Invariantes
- Regla innegociable: `quantity > 0` ($value \ge 1$).
- Se rechaza cualquier valor $\le 0$ (cero o negativos) lanzando `InvalidQuantityException`.

### 4.4. Operaciones Públicas Justificadas
Para soportar las necesidades de gestión de ejemplares (reabastecimiento y cálculo de copias):
1. `public function value(): int`: Retorna el valor escalar entero.
2. `public function add(Quantity $other): Quantity`: Retorna una **nueva** instancia de `Quantity` con la suma de ambos valores.
   - Si la suma excede `PHP_INT_MAX` (desbordamiento), se detecta y lanza `InvalidQuantityException`.
   - Garantiza inmutabilidad: el objeto original no cambia.
3. `public function equals(Quantity $other): bool`: Compara igualdad de valor escalar.

*Nota de alcance:* No se implementan operaciones de resta, multiplicación o división en este objeto de valor, ya que la sustracción de copias disponibles puede llegar a 0 (lo cual corresponde al estado del agregado `Book` y su columna `available_copies`, no a una `Quantity` positiva).

### 4.5. Pruebas Requeridas para `Quantity`
| # | Entrada / Operación | Resultado Esperado | Excepción |
|---|---|---|---|
| P-QTY-01 | `1` (mínimo positivo) | Instancia creada, `value() === 1` | Ninguna |
| P-QTY-02 | `50` (entero positivo) | Instancia creada, `value() === 50` | Ninguna |
| P-QTY-03 | `0` (cero) | Rechazado por no ser positivo | `InvalidQuantityException` |
| P-QTY-04 | `-1` (negativo) | Rechazado por ser negativo | `InvalidQuantityException` |
| P-QTY-05 | `1.5` o `1.0` (flotante) | Rechazado por tipado estricto PHP | `TypeError` |
| P-QTY-06 | `"5"` (cadena numérica) | Rechazado por tipado estricto PHP | `TypeError` |
| P-QTY-07 | `true` (booleano) | Rechazado por tipado estricto PHP | `TypeError` |
| P-QTY-08 | `Quantity(5)->add(Quantity(3))` | Retorna nuevo `Quantity(8)`; original se mantiene en `5` | Ninguna |
| P-QTY-09 | `Quantity(5)->equals(Quantity(5))` | Retorna `true` | Ninguna |
| P-QTY-10 | Desbordamiento (`PHP_INT_MAX + 1`) | Rechazado | `InvalidQuantityException` |

---

## 5. Objeto de Valor `DateRange`

### 5.1. Propósito
Representar un intervalo temporal inmutable cerrado `[startDate, endDate]` para filtros de fecha y reportes (ej. HU-10: reporte de más prestados).

### 5.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\DateRange`
- **Constructor:** `public function __construct(DateTimeImmutable $startDate, DateTimeImmutable $endDate)`
- **Tipo de entrada:** Exclusivamente instancias de `DateTimeImmutable`. No se aceptan instancias mutables de `DateTime` ni cadenas en el constructor para evitar acoplamientos o mutaciones externas inesperadas.

### 5.3. Invariante Temporal y Modelo de Intervalo
- **Invariante fundamental:** `startDate <= endDate`.
- **Intervalo cerrado:** `[startDate, endDate]`.
  - Si `startDate < endDate`: Rango válido.
  - Si `startDate == endDate`: Rango de duración cero válido (mismo instante temporal).
  - Si `endDate < startDate`: Rango inválido; lanza `InvalidDateRangeException` (alineado con CA-10.2).

### 5.4. Normalización a UTC (Artículo VI y RN-6)
- Toda fecha se normaliza a la zona horaria UTC (`new DateTimeZone('UTC')`) al ingresar.
- Si las fechas se proporcionan con zonas horarias distintas (ej. `UTC-5` y `UTC+2`), se convierten a su instante UTC equivalente antes de comparar y almacenar. La comparación temporal evalúa el instante cronológico real (`getTimestamp()`).
- Los métodos de acceso `startDate()` y `endDate()` devuelven instancias en UTC.
- **Política de fechas sin zona horaria:** `DateTimeImmutable` en PHP siempre posee una zona horaria asociada (o asume UTC por configuración). Para garantizar determinismo en el dominio, todo objeto recibido es formalmente transicionado a UTC mediante `$date->setTimezone(new DateTimeZone('UTC'))`.

### 5.5. Métodos Públicos Justificados
- `public function startDate(): DateTimeImmutable`: Retorna el inicio normalizado en UTC.
- `public function endDate(): DateTimeImmutable`: Retorna el fin normalizado en UTC.
- `public function includes(DateTimeImmutable $date): bool`: Determina si un instante está comprendido en el intervalo cerrado (`$start <= $dateUTC && $dateUTC <= $end`).
- `public function equals(DateRange $other): bool`: Compara igualdad de ambos extremos en UTC.

### 5.6. Pruebas Requeridas para `DateRange`
| # | Entrada / Operación | Resultado Esperado | Excepción |
|---|---|---|---|
| P-DR-01 | `2026-01-01 10:00:00` y `2026-01-02 10:00:00` (UTC) | Instancia creada correctamente | Ninguna |
| P-DR-02 | `2026-01-01 10:00:00` y `2026-01-01 10:00:00` (mismo instante) | Válido (duración cero permitida) | Ninguna |
| P-DR-03 | `2026-01-02 10:00:00` y `2026-01-01 10:00:00` (fin anterior a inicio) | Rechazado | `InvalidDateRangeException` |
| P-DR-04 | Extremos con distintas zonas horarias pero fin posterior en tiempo absoluto | Convertidos a UTC; rango válido | Ninguna |
| P-DR-05 | Extremos con distintas representaciones que son el mismo instante absoluto (ej. 15:00 UTC vs 10:00 UTC-5) | Válido, ambos normalizados a 15:00 UTC | Ninguna |
| P-DR-06 | Inmutabilidad de los argumentos | Modificar fechas externas o llamar métodos no muta el rango | Ninguna |
| P-DR-07 | `includes()` con fecha dentro del rango | Retorna `true` | Ninguna |
| P-DR-08 | `includes()` con fecha exactamente igual a `startDate` o `endDate` | Retorna `true` (intervalo cerrado) | Ninguna |
| P-DR-09 | `includes()` con fecha fuera del rango | Retorna `false` | Ninguna |

---

## 6. Suite de Pruebas Unitarias

- **Ubicación:** `test-manager-library-3413974-api/tests/Unit/Domain/ValueObjectsTest.php`
- **Características:**
  - Pruebas unitarias puras que heredan de `PHPUnit\Framework\TestCase`.
  - Cero interacción con base de datos, Docker daemon o contenedor web.
  - Ejecución en menos de 100 ms.
- **Comando de ejecución exacto en Docker (según configuración de T-01 y T-02):**
  ```bash
  docker compose run --rm test php artisan test --filter=ValueObjectsTest
  ```

---

## 7. Matriz de Trazabilidad de Requisitos

| ID Regla | Objeto | Condición / Invariante | Resultado Esperado | Excepción | Prueba Asociada |
|---|---|---|---|---|---|
| **TR-ISBN-01** | `Isbn` | 13 dígitos numéricos válidos | Instancia válida | Ninguna | `P-ISBN-01` |
| **TR-ISBN-02** | `Isbn` | Entrada con guiones o espacios exteriores | Normalizado a 13 dígitos puros | Ninguna | `P-ISBN-02`, `P-ISBN-03` |
| **TR-ISBN-03** | `Isbn` | Longitud $\ne 13$ o caracteres no numéricos | Rechazo inmediato | `InvalidIsbnException` | `P-ISBN-05` a `P-ISBN-11` |
| **TR-QTY-01** | `Quantity` | Entero positivo ($value \ge 1$) | Instancia válida | Ninguna | `P-QTY-01`, `P-QTY-02` |
| **TR-QTY-02** | `Quantity` | Entero $\le 0$ (cero o negativo) | Rechazo inmediato | `InvalidQuantityException` | `P-QTY-03`, `P-QTY-04` |
| **TR-QTY-03** | `Quantity` | Operación `add()` inmutable | Nueva instancia con suma | Ninguna / `InvalidQuantityException` (overflow) | `P-QTY-08`, `P-QTY-10` |
| **TR-DR-01** | `DateRange` | `startDate <= endDate` | Instancia válida en UTC | Ninguna | `P-DR-01`, `P-DR-02` |
| **TR-DR-02** | `DateRange` | `endDate < startDate` | Rechazo inmediato | `InvalidDateRangeException` | `P-DR-03` |
| **TR-DR-03** | `DateRange` | Distintas zonas horarias en entrada | Normalización a UTC preservando instante | Ninguna | `P-DR-04`, `P-DR-05` |
| **TR-DR-04** | `DateRange` | Verificación de inclusión cerrada | Evalúa pertenencia $[start, end]$ | Ninguna | `P-DR-07`, `P-DR-08`, `P-DR-09` |

---

## 8. Procedimiento de Verificación y Entrega
1. Se mantiene el repositorio `api` 100% inalterado durante esta fase.
2. Esta especificación cubre cada aspecto solicitado sin ambigüedades técnicas ni dependencias de frameworks.
3. Se somete a la revisión técnica externa de ChatGPT para su aprobación antes de proceder a la fase de programación.
