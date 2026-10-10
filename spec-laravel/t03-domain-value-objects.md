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

### 2.4. Blindaje contra Tipado Débil y Rechazo Estricto de Tipos (Defensa en Profundidad)
- **El problema del tipado débil en PHP:** En PHP, la directiva `declare(strict_types=1);` posee alcance léxico por archivo (*file-scoped*). Solo afecta a las invocaciones realizadas desde el archivo donde está declarada. Si un llamador (controlador, servicio o consumidor externo) omite `declare(strict_types=1);`, PHP activa automáticamente su modo de tipado coercitivo débil (*weak/coercive typing*).
- **Vulnerabilidad de coerción con hints nativos escalares:**
  - Si un constructor declara `string $value` y un llamador en tipado débil pasa un entero (`9789584276971`), PHP lo coerciona silenciosamente a `"9789584276971"`. Si el entero comenzaba con `0` (ej. `0123456789012`), el cero inicial se pierde antes de que el objeto reciba el valor.
  - Si un constructor declara `int $value` y un llamador en tipado débil pasa una cadena numérica (`"5"`) o un flotante entero (`5.0`), PHP lo coerciona silenciosamente al entero `5`.
- **Mecanismo de blindaje en el dominio:**
  1. Todos los archivos de las clases del dominio mantienen `declare(strict_types=1);`.
  2. Para anular cualquier intento de coerción en el sitio de llamada (*call-site*), los constructores declaran sus parámetros como `mixed` (ej. `public function __construct(mixed $value)` en `Isbn` y `Quantity`, y `public function __construct(mixed $startDate, mixed $endDate)` en `DateRange`).
  3. Al tipar los parámetros como `mixed`, el motor de PHP **nunca aplica coerción de tipos**, sin importar si el llamador tiene o no activo `declare(strict_types=1)`. El argumento entra con su tipo primitivo real intacto.
  4. En la primera línea del cuerpo del constructor, se evalúa una aserción defensiva estricta de tipo en tiempo de ejecución:
     - `Isbn`: `if (!is_string($value)) { throw new \TypeError('ISBN debe ser una cadena de texto (string).'); }`
     - `Quantity`: `if (!is_int($value)) { throw new \TypeError('La cantidad debe ser un número entero (int).'); }`
     - `DateRange`: `if (!$startDate instanceof \DateTimeImmutable || !$endDate instanceof \DateTimeImmutable) { throw new \TypeError('Las fechas deben ser instancias de DateTimeImmutable.'); }`
  5. Si el tipo no coincide estrictamente con el requerido, se lanza `\TypeError` antes de cualquier procesamiento o validación de reglas de negocio.
  6. Esto asegura un comportamiento idéntico e infranqueable tanto si el llamador utiliza tipado estricto como si utiliza tipado débil.

---

## 3. Objeto de Valor `Isbn`

### 3.1. Propósito
Representar el identificador ISBN-13 de un libro del catálogo mediante un objeto de valor inmutable y normalizado.

### 3.2. Contrato y Tipos
- **Namespace:** `Library\Domain\ValueObjects\Isbn`
- **Constructor:** `public function __construct(mixed $value)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente `string` validado defensivamente (`is_string($value)`). Cualquier tipo no-string (`int`, `float`, `bool`, `array`, `object`, `null`) es rechazado de inmediato con `\TypeError`, impidiendo cualquier coerción silenciosa (por ejemplo, enteros como `9789584276971`) en contextos de tipado débil.

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
- **Constructor:** `public function __construct(mixed $value)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente `int` validado defensivamente (`is_int($value)`). Cualquier tipo no-int (`string` numérico como `"1"`, `float` `1.0`, `bool`, `array`, `object`, `null`) es rechazado de inmediato con `\TypeError`, impidiendo la coerción automática de cadenas numéricas a enteros en contextos de tipado débil.

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
- **Constructor:** `public function __construct(mixed $startDate, mixed $endDate)` con `declare(strict_types=1);`.
- **Tipo de entrada:** Exclusivamente instancias de `DateTimeImmutable` validadas defensivamente (`$startDate instanceof DateTimeImmutable && $endDate instanceof DateTimeImmutable`). Cadenas de texto (`"2026-01-01"`), enteros, instancias mutables de `DateTime` o `null` son rechazados de inmediato con `\TypeError`.

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
- **Estrategia para pruebas de tipado débil y rechazo de tipos:**  
  Para certificar que el blindaje contra tipado débil funciona de forma inequívoca en tiempo de ejecución, la suite incluye pruebas unitarias que pasan argumentos de tipos incompatibles (enteros a `Isbn`, cadenas o flotantes a `Quantity`, cadenas a `DateRange`) verificando el lanzamiento de `\TypeError`. Adicionalmente, se comprueba la invocación simulando llamadores sin `declare(strict_types=1);` (mediante invocadores auxiliares o funciones sin tipado estricto) para verificar empíricamente que PHP no aplica conversión ni coerción silenciosa sobre los argumentos recibidos como `mixed`.
- **Comando exacto de ejecución en Docker:**  
  De acuerdo con la configuración real de `docker-compose.yml` aprobada en T-01 y T-02 (donde el servicio de pruebas es `test` con `command: ["php", "artisan", "test"]`), el comando exacto para ejecutar esta suite unitaria es:
  ```bash
  docker compose run --rm test php artisan test tests/Unit/Domain/ValueObjectsTest.php
  ```

---

## 7. Catálogo Completo de Casos de Prueba (37 Casos Exhaustivos)

### A. Casos de Prueba para `Isbn` (15 Casos: P-ISBN-01 a P-ISBN-15)
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
| **P-ISBN-14** | `12.5` (float), `true` (bool), `['9789584276971']` (array), `null` | Rechazado por validación estricta de tipo | `TypeError` |
| **P-ISBN-15** | Invocación en contexto de tipado débil con entero `9789584276971` | Rechazado sin coerción silenciosa a `string` | `TypeError` |

### B. Casos de Prueba para `Quantity` (12 Casos: P-QTY-01 a P-QTY-12)
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
| **P-QTY-11** | `5.5` (float), `true` (bool), `[5]` (array), `null` | Rechazado por validación estricta de tipo | `TypeError` |
| **P-QTY-12** | Invocación en contexto de tipado débil con cadena numérica `"10"` o `10.0` | Rechazado sin coerción silenciosa a `int` | `TypeError` |

### C. Casos de Prueba para `DateRange` (10 Casos: P-DR-01 a P-DR-10)
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
| **P-DR-10** | Cadenas (`"2026-01-01"`), objetos mutables (`new DateTime()`), enteros o `null` | Rechazado por validación estricta de tipo | `TypeError` |

*Total reconciliado:* Exactamente **37 casos de prueba** (15 de ISBN + 12 de Quantity + 10 de DateRange), cubriendo cada método público, invariante, blindaje contra tipado débil y excepción.

---

## 8. Matriz de Trazabilidad Cruzada

| ID Regla | Objeto | Regla / Invariante | Condición / Método | Excepción | Casos de Prueba Asociados |
|---|---|---|---|---|---|
| **TR-ISBN-01** | `Isbn` | 13 dígitos numéricos válidos | Entrada válida o con ceros | Ninguna | `P-ISBN-01`, `P-ISBN-04` |
| **TR-ISBN-02** | `Isbn` | Normalización de separadores y espacios | Entrada con guiones o espacios exteriores | Ninguna | `P-ISBN-02`, `P-ISBN-03` |
| **TR-ISBN-03** | `Isbn` | Rechazo de formato no numérico o longitud errónea | Longitud $\ne 13$, letras, signos, espacios interiores | `InvalidIsbnException` | `P-ISBN-05`, `P-ISBN-06`, `P-ISBN-07`, `P-ISBN-08`, `P-ISBN-09`, `P-ISBN-10`, `P-ISBN-11` |
| **TR-ISBN-04** | `Isbn` | Igualdad y representación textual | Métodos `equals()` y `__toString()` | Ninguna | `P-ISBN-12`, `P-ISBN-13` |
| **TR-ISBN-05** | `Isbn` | Blindaje de tipo y rechazo ante tipado débil | Entrada no-string (`int`, `float`, `bool`, etc.) o llamada débil | `TypeError` | `P-ISBN-14`, `P-ISBN-15` |
| **TR-QTY-01** | `Quantity` | Cantidad estrictamente positiva | Enteros $value \ge 1$ | Ninguna | `P-QTY-01`, `P-QTY-02` |
| **TR-QTY-02** | `Quantity` | Rechazo de valores $\le 0$ | Cero o enteros negativos | `InvalidQuantityException` | `P-QTY-03`, `P-QTY-04`, `P-QTY-05` |
| **TR-QTY-03** | `Quantity` | Operación aritmética inmutable | Método `add()` retornando nueva instancia | Ninguna | `P-QTY-06`, `P-QTY-07` |
| **TR-QTY-04** | `Quantity` | Detección previa de desbordamiento | `add()` con suma $> PHP\_INT\_MAX$ | `InvalidQuantityException` | `P-QTY-08` |
| **TR-QTY-05** | `Quantity` | Igualdad por valor | Método `equals()` | Ninguna | `P-QTY-09`, `P-QTY-10` |
| **TR-QTY-06** | `Quantity` | Blindaje de tipo y rechazo ante tipado débil | Entrada no-int (`string`, `float`, `bool`, etc.) o llamada débil | `TypeError` | `P-QTY-11`, `P-QTY-12` |
| **TR-DR-01** | `DateRange` | Invariante temporal $startDate \le endDate$ | Rango con fin $\ge$ inicio (incluye duración 0) | Ninguna | `P-DR-01`, `P-DR-02` |
| **TR-DR-02** | `DateRange` | Rechazo de rango invertido | $endDate < startDate$ | `InvalidDateRangeException` | `P-DR-03` |
| **TR-DR-03** | `DateRange` | Normalización determinista a UTC | Extremos con zonas horarias arbitrarias | Ninguna | `P-DR-04`, `P-DR-05`, `P-DR-06` |
| **TR-DR-04** | `DateRange` | Inclusión cerrada e igualdad | Métodos `includes()` y `equals()` | Ninguna | `P-DR-07`, `P-DR-08`, `P-DR-09` |
| **TR-DR-05** | `DateRange` | Blindaje de tipo contra tipos no compatibles | Entrada no-`DateTimeImmutable` (`string`, `DateTime`, etc.) | `TypeError` | `P-DR-10` |
