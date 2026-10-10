# T-03 — Especificación Técnica y Criterios de Aceptación de los Objetos de Valor

## 1. Identificador y Alcance de la Tarea
- **ID de la Tarea:** T-03 (según `spec-.net/tasks.md` y `spec-python/tasks.md`).
- **Propósito:** Documentar exhaustivamente las reglas de negocio, contratos, invariantes, excepciones y casos de prueba para los tres objetos de valor fundamentales del dominio:
  - `Isbn`
  - `Quantity`
  - `DateRange`
- **Ámbito:** Exclusivamente el núcleo de dominio puro (`src/Domain/ValueObjects` y `src/Domain/Exceptions`) en `test-manager-library-3413974-api`. Sin dependencias de Laravel, HTTP, ORM ni servicios de infraestructura (Artículo I de `constitution.md`).

---

## 2. Reglas Generales de los Objetos de Valor

### 2.1. Inmutabilidad y Encapsulamiento
- Implementación como clases `final readonly class` en PHP 8.2+.
  - `final`: Impide que otras clases hereden de los objetos.
  - `readonly`: Impide la reasignación o mutación de sus propiedades tras su inicialización en el constructor.
- Validación inmediata en la construcción: Ninguna instancia puede existir en un estado inválido o inconsistente. La instanciación falla lanzando la excepción de dominio correspondiente antes de completar la construcción.
- Todas las operaciones que generen nuevos valores retornan una nueva instancia inmutable y no modifican el objeto original.
- Nomenclatura en inglés para clases y métodos (Artículo XI). Mensajes de excepción descriptivos en español.

### 2.2. Independencia Absoluta del Framework (Artículo I)
- Ubicación estricta: `src/Domain/ValueObjects/` y `src/Domain/Exceptions/` bajo el namespace raíz `Library\Domain\`.
- Cero dependencias de Laravel (`Illuminate\*`), Eloquent, HTTP, bases de datos o librerías de terceros (POPO puro).
- Solo tipos estándar de PHP (`string`, `int`, `bool`, `DateTimeImmutable`, `DateTimeZone`, `DomainException`).

### 2.3. Jerarquía y Contrato de Excepciones
Para mantener una jerarquía inequívoca, limpia y compatible con las convenciones del proyecto:
- Se define una clase base abstracta de dominio: `Library\Domain\Exceptions\DomainException` que extiende de la clase nativa `\DomainException` de PHP (SPL).
- Todas las excepciones de objetos de valor extienden de `Library\Domain\Exceptions\DomainException`:
  1. `InvalidIsbnException`: Lanzada ante cualquier incumplimiento del formato o caracteres permitidos del ISBN.
  2. `InvalidQuantityException`: Lanzada ante valores no positivos ($\le 0$) o desbordamiento aritmético.
  3. `InvalidDateRangeException`: Lanzada ante rangos temporales invertidos ($fin < inicio$).
- Cada excepción incluye un constructor que define un mensaje claro en español sobre la causa del rechazo.

---

## 3. Objeto de Valor `Isbn`

### 3.1. Propósito
Representar el identificador ISBN-13 de un libro del catálogo mediante un objeto de valor inmutable y normalizado.

### 3.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\Isbn`
- **Constructor:** `public function __construct(string $value)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente `string`. El tipado estricto rechaza enteros, flotantes, booleanos, arreglos y nulos (lanzan `TypeError` a nivel de lenguaje).

### 3.3. Formato e Invariantes
- **Expresión de formato de referencia:** `^[0-9]{13}$` (exactamente 13 dígitos numéricos ASCII entre `0` y `9`).
- **Reglas de rechazo (lanzan `InvalidIsbnException`):**
  - Cadenas vacías o compuestas únicamente por espacios.
  - Cadenas con longitud menor a 13 dígitos (ej. 10 o 12 dígitos).
  - Cadenas con longitud mayor a 13 dígitos (ej. 14 dígitos).
  - Caracteres alfabéticos (ej. `"978123456789X"` o `"978012345678a"`).
  - Caracteres especiales o signos (ej. `"+9781234567890"`, `"-9781234567890"`).
  - Espacios interiores (ej. `"978 1234567890"`).
  - Notación científica o caracteres Unicode fuera del rango ASCII `0-9`.
- **Ceros iniciales:** Se preservan y son válidos siempre que completen exactamente 13 dígitos (ej. `"0123456789012"`).

### 3.4. Normalización
- **Política aprobada:**
  1. Se recortan espacios exteriores al inicio y al final mediante `trim($value)`.
  2. Se remueven guiones separadores mediante `str_replace('-', '', ...)`.
  3. La cadena resultante se evalúa contra `/^[0-9]{13}$/`.
- El valor normalizado resultante es el que se almacena internamente y se retorna por `value()`.
- Cualquier entrada que tras remover guiones y espacios exteriores no contenga exactamente 13 dígitos numéricos ASCII es rechazada con `InvalidIsbnException`.

### 3.5. Decisión Técnica: Formato vs. Dígito de Control
- **Decisión cerrada (Alternativa A - Formato Estricto):** Se valida estrictamente el formato de 13 dígitos numéricos ASCII (`^[0-9]{13}$`). No se impone el algoritmo ponderado mod 10 del ISBN-13, en estricta conformidad con `spec.md` (CA-03.2: *"El ISBN tiene 13 dígitos"*), `data-model.md` (§3: *"isbn char(13): Solo dígitos"*) y `tasks.md` (T-03: *"ISBN de 13 dígitos"*).

### 3.6. Métodos Públicos
- `public function value(): string`: Retorna la cadena limpia de 13 dígitos.
- `public function equals(Isbn $other): bool`: Compara igualdad de valor (`$this->value === $other->value()`).
- `public function __toString(): string`: Retorna el valor normalizado.

---

## 4. Objeto de Valor `Quantity`

### 4.1. Propósito
Representar una cantidad de copias o ejemplares estrictamente positiva dentro del dominio ($quantity \ge 1$).

### 4.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\Quantity`
- **Constructor:** `public function __construct(int $value)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente `int`. Flotantes (`1.0`), cadenas numéricas (`"1"`), booleanos y nulos son rechazados por tipado estricto (`TypeError`).

### 4.3. Invariantes
- Invariante obligatoria: `quantity > 0` ($value \ge 1$).
- Valores $\le 0$ (cero o enteros negativos) son rechazados lanzando `InvalidQuantityException`.

### 4.4. Operaciones Públicas y Detección de Desbordamiento
1. `public function value(): int`: Retorna el valor escalar entero.
2. `public function add(Quantity $other): Quantity`: Retorna una nueva instancia con la suma de ambos valores.
   - **Condición de desbordamiento evaluada antes de sumar:**  
     Para prevenir desbordamientos de enteros en PHP (que convertirían el valor a `float` silenciosamente), se valida preventivamente antes de la operación:  
     `if ($other->value() > PHP_INT_MAX - $this->value) { throw new InvalidQuantityException("Desbordamiento aritmético al sumar cantidades."); }`
   - Garantiza inmutabilidad estricta: el objeto original receptor no se modifica.
3. `public function equals(Quantity $other): bool`: Retorna `true` si ambos objetos poseen el mismo valor numérico escalar.

*Delimitación de alcance:* No se incluyen resta, multiplicación ni división. El decremento de copias que alcanza 0 es gestionado en el estado del agregado `Book` (`available_copies`), no mediante el objeto `Quantity`.

---

## 5. Objeto de Valor `DateRange`

### 5.1. Propósito
Representar un intervalo temporal cerrado `[startDate, endDate]` inmutable, normalizado y coherente para filtros y reportes (HU-10).

### 5.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\DateRange`
- **Constructor:** `public function __construct(DateTimeImmutable $startDate, DateTimeImmutable $endDate)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente instancias de `DateTimeImmutable`. No se admiten cadenas ni instancias mutables de `DateTime`.

### 5.3. Invariante Temporal y Modelo de Intervalo
- Invariante fundamental: El instante cronológico de inicio debe ser anterior o igual al instante de fin ($startDate \le endDate$).
- Intervalo cerrado: $[startDate, endDate]$. Ambos extremos pertenecen al intervalo.
- Si $startDate == endDate$, el rango de duración cero es válido.
- Si $endDate < startDate$, se rechaza lanzando `InvalidDateRangeException` (alineado con CA-10.2).

### 5.4. Tratamiento Preciso de Fechas y Normalización a UTC (RN-6)
- **Distinción entre interpretación y conversión:**
  1. En PHP, toda instancia de `DateTimeImmutable` posee un instante absoluto inequívoco (obtenible mediante `getTimestamp()` / microsegundos) y una zona horaria asociada (`getTimezone()`).
  2. Si una fecha fue construida sin especificar zona horaria en el código cliente, PHP le asigna la zona horaria del entorno (`date.timezone`, por defecto UTC en nuestro contenedor Docker).
  3. **Normalización determinista:** El constructor de `DateRange` no asume representaciones locales. Transiciona inequívocamente cada extremo a UTC preservando exactamente su instante cronológico mediante:  
     `$startUtc = $startDate->setTimezone(new DateTimeZone('UTC'));`  
     `$endUtc = $endDate->setTimezone(new DateTimeZone('UTC'));`
  4. La evaluación de la invariante ($startUtc \le endUtc$) y el almacenamiento se realizan sobre los objetos normalizados a UTC.
  5. Los métodos de acceso `startDate()` y `endDate()` devuelven siempre instancias cuya zona horaria es estrictamente UTC (`DateTimeZone('UTC')`).

### 5.5. Métodos Públicos Justificados
- `public function startDate(): DateTimeImmutable`: Retorna el inicio normalizado en UTC.
- `public function endDate(): DateTimeImmutable`: Retorna el fin normalizado en UTC.
- `public function includes(DateTimeImmutable $date): bool`: Evalúa si un instante pertenece al intervalo cerrado. Normaliza `$date` a UTC y comprueba `$this->startDate <= $dateUtc && $dateUtc <= $this->endDate`.
- `public function equals(DateRange $other): bool`: Retorna `true` si ambos rangos coinciden exactamente en sus instantes UTC de inicio y fin.

---

## 6. Suite de Pruebas Unitarias y Ejecución Docker

- **Ubicación:** `test-manager-library-3413974-api/tests/Unit/Domain/ValueObjectsTest.php`
- **Características:**
  - Pruebas unitarias puras que heredan de `PHPUnit\Framework\TestCase`.
  - Cero interacción con base de datos, extensiones externas o controladores web.
- **Comando exacto de ejecución en Docker:**  
  De acuerdo con la configuración real de `docker-compose.yml` aprobada en T-01 y T-02 (donde el servicio de pruebas es `test` con `command: ["php", "artisan", "test"]`), el comando exacto para ejecutar esta suite unitaria es:
  ```bash
  docker compose run --rm test php artisan test tests/Unit/Domain/ValueObjectsTest.php
  ```

---

## 7. Catálogo Completo de Casos de Prueba (32 Casos Exhaustivos)

### A. Casos de Prueba para `Isbn` (13 Casos: P-ISBN-01 a P-ISBN-13)
| ID | Entrada | Resultado Esperado | Excepción |
|---|---|---|---|
| **P-ISBN-01** | `"9789584276971"` | Instancia creada; `value() === "9789584276971"` | Ninguna |
| **P-ISBN-02** | `"978-958-42-7697-1"` | Normalizado correctamente a `"9789584276971"` | Ninguna |
| **P-ISBN-03** | `"  9789584276971  "` | Normalizado eliminando espacios exteriores | Ninguna |
| **P-ISBN-04** | `"0123456789012"` | Válido; preserva ceros iniciales | Ninguna |
| **P-ISBN-05** | `""` (cadena vacía o espacios) | Rechazado | `InvalidIsbnException` |
| **P-ISBN-06** | `"978123456789"` (12 dígitos) | Rechazado por longitud insuficiente | `InvalidIsbnException` |
| **P-ISBN-07** | `"97812345678901"` (14 dígitos) | Rechazado por longitud excesiva | `InvalidIsbnException` |
| **P-ISBN-08** | `"978123456789X"` | Rechazado por caracter alfabético | `InvalidIsbnException` |
| **P-ISBN-09** | `"978-1234-5678-A"` | Rechazado por letra tras normalizar | `InvalidIsbnException` |
| **P-ISBN-10** | `"978 1234 56789"` | Rechazado por espacio interior | `InvalidIsbnException` |
| **P-ISBN-11** | `"+9781234567890"` | Rechazado por signos no numéricos | `InvalidIsbnException` |
| **P-ISBN-12** | Comparación `equals()` | `true` para valores equivalentes, `false` para diferentes | Ninguna |
| **P-ISBN-13** | Representación `__toString()` | Coincide con `value()` | Ninguna |

### B. Casos de Prueba para `Quantity` (10 Casos: P-QTY-01 a P-QTY-10)
| ID | Entrada / Operación | Resultado Esperado | Excepción |
|---|---|---|---|
| **P-QTY-01** | `1` (mínimo positivo) | Instancia creada; `value() === 1` | Ninguna |
| **P-QTY-02** | `100` (entero positivo) | Instancia creada; `value() === 100` | Ninguna |
| **P-QTY-03** | `0` (cero) | Rechazado; no es estrictamente positivo | `InvalidQuantityException` |
| **P-QTY-04** | `-1` (negativo) | Rechazado; no es estrictamente positivo | `InvalidQuantityException` |
| **P-QTY-05** | `-50` (negativo grande) | Rechazado | `InvalidQuantityException` |
| **P-QTY-06** | Operación `add(Quantity(3))` sobre `Quantity(5)` | Retorna nueva instancia `Quantity(8)` | Ninguna |
| **P-QTY-07** | Inmutabilidad de `add()` | El objeto original mantiene su valor `5` inalterado | Ninguna |
| **P-QTY-08** | Detección previa de desbordamiento | `Quantity(PHP_INT_MAX)->add(Quantity(1))` lanza excepción preventiva | `InvalidQuantityException` |
| **P-QTY-09** | Comparación `equals()` idéntico | Retorna `true` para cantidades iguales | Ninguna |
| **P-QTY-10** | Comparación `equals()` distinto | Retorna `false` para cantidades diferentes | Ninguna |

### C. Casos de Prueba para `DateRange` (9 Casos: P-DR-01 a P-DR-09)
| ID | Entrada / Operación | Resultado Esperado | Excepción |
|---|---|---|---|
| **P-DR-01** | Inicio `2026-01-01 10:00:00` y Fin `2026-01-02 10:00:00` (UTC) | Instancia creada exitosamente | Ninguna |
| **P-DR-02** | Inicio y Fin con el mismo instante temporal | Válido; rango de duración cero permitido | Ninguna |
| **P-DR-03** | Fin anterior a Inicio ($endDate < startDate$) | Rechazado | `InvalidDateRangeException` |
| **P-DR-04** | Zonas horarias distintas pero fin posterior absoluto | Normalizados a UTC; rango válido | Ninguna |
| **P-DR-05** | Distintas representaciones de zona que son el mismo instante absoluto (ej. 15:00 UTC vs 10:00 UTC-5) | Válido; ambos convertidos al mismo instante UTC | Ninguna |
| **P-DR-06** | **Comprobación explícita de normalización e inmutabilidad:** pasar objetos con zona no-UTC (ej. America/Bogota), verificar que `startDate()` y `endDate()` devuelven zona UTC, con el mismo timestamp Unix exacto, y que los objetos originales no mutan | Normalización rigurosa a UTC e instantes conservados | Ninguna |
| **P-DR-07** | `includes()` dentro del rango | Retorna `true` para instantes intermedios | Ninguna |
| **P-DR-08** | `includes()` en los límites exactos | Retorna `true` en inicio y en fin (intervalo cerrado) | Ninguna |
| **P-DR-09** | `includes()` fuera del rango y `equals()` | `includes()` retorna `false`; `equals()` valida igualdad UTC | Ninguna |

*Total reconciliado:* Exactamente **32 casos de prueba** (13 de ISBN + 10 de Quantity + 9 de DateRange), cubriendo cada método público, invariante y excepción.

---

## 8. Matriz de Trazabilidad Cruzada

| ID Regla | Objeto | Regla / Invariante | Condición / Método | Excepción | Casos de Prueba Asociados |
|---|---|---|---|---|---|
| **TR-ISBN-01** | `Isbn` | 13 dígitos numéricos válidos | Entrada válida o con ceros | Ninguna | `P-ISBN-01`, `P-ISBN-04` |
| **TR-ISBN-02** | `Isbn` | Normalización de separadores y espacios | Entrada con guiones o espacios exteriores | Ninguna | `P-ISBN-02`, `P-ISBN-03` |
| **TR-ISBN-03** | `Isbn` | Rechazo de formato no numérico o longitud errónea | Longitud $\ne 13$, letras, signos, espacios interiores | `InvalidIsbnException` | `P-ISBN-05`, `P-ISBN-06`, `P-ISBN-07`, `P-ISBN-08`, `P-ISBN-09`, `P-ISBN-10`, `P-ISBN-11` |
| **TR-ISBN-04** | `Isbn` | Igualdad y representación textual | Métodos `equals()` y `__toString()` | Ninguna | `P-ISBN-12`, `P-ISBN-13` |
| **TR-QTY-01** | `Quantity` | Cantidad estrictamente positiva | Enteros $value \ge 1$ | Ninguna | `P-QTY-01`, `P-QTY-02` |
| **TR-QTY-02** | `Quantity` | Rechazo de valores $\le 0$ | Cero o enteros negativos | `InvalidQuantityException` | `P-QTY-03`, `P-QTY-04`, `P-QTY-05` |
| **TR-QTY-03** | `Quantity` | Operación aritmética inmutable | Método `add()` retornando nueva instancia | Ninguna | `P-QTY-06`, `P-QTY-07` |
| **TR-QTY-04** | `Quantity` | Detección previa de desbordamiento | `add()` con suma $> PHP\_INT\_MAX$ | `InvalidQuantityException` | `P-QTY-08` |
| **TR-QTY-05** | `Quantity` | Igualdad por valor | Método `equals()` | Ninguna | `P-QTY-09`, `P-QTY-10` |
| **TR-DR-01** | `DateRange` | Invariante temporal $startDate \le endDate$ | Rango con fin $\ge$ inicio (incluye duración 0) | Ninguna | `P-DR-01`, `P-DR-02` |
| **TR-DR-02** | `DateRange` | Rechazo de rango invertido | $endDate < startDate$ | `InvalidDateRangeException` | `P-DR-03` |
| **TR-DR-03** | `DateRange` | Normalización determinista a UTC | Extremos con zonas horarias arbitrarias | Ninguna | `P-DR-04`, `P-DR-05`, `P-DR-06` |
| **TR-DR-04** | `DateRange` | Inclusión cerrada e igualdad | Métodos `includes()` y `equals()` | Ninguna | `P-DR-07`, `P-DR-08`, `P-DR-09` |
