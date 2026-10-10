# T-04 a T-06 — Especificación de Entidades y Reglas de Dominio

## 1. Identificadores y Alcance
- **Tareas Cubiertas:**
  - **T-04:** Dominio `Book` (`withdraw()`, `restock()`, `adjustTotalCopies()`, baja lógica).
  - **T-05:** Dominio `Loan` y `LoanItem` (14 días en UTC, 1 a 3 libros sin duplicados, estado calculado `open`, `overdue`, `closed`, devolución).
  - **T-06:** Dominio `Member` y `User` (documento único, normalización de nombre de usuario en minúsculas, roles válidos).
- **Ubicación en el repositorio API:** `src/Domain/Entities/` y `src/Domain/Exceptions/`.
- **Independencia Arquitectónica:** Clases POPO puras en PHP 8.2+ sin dependencias de Laravel, Eloquent o infraestructura (Artículo I).

---

## 2. Entidad `Book` (T-04)

### 2.1. Invariantes
1. **Rango de Disponibles (RN-1):** En todo momento se cumple `0 <= available_copies <= total_copies`.
2. **Retiro (`withdraw()`):** Reduce en 1 los ejemplares disponibles. Si `available_copies <= 0`, lanza `BookNotAvailableException` (409).
3. **Reabastecimiento / Devolución (`restock()`):** Aumenta en 1 los ejemplares disponibles. Si `available_copies >= total_copies`, lanza `ValidationException` (422).
4. **Ajuste de Total (`adjustTotalCopies()`):** Si el total de copias se modifica, los ejemplares disponibles se ajustan en la misma diferencia ($disponibles_{nuevo} = disponibles_{actual} + (total_{nuevo} - total_{actual})$). Si el resultado dejara los disponibles en negativo ($< 0$), se rechaza lanzando `ValidationException` (422).
5. **Baja Lógica (`deactivate()`):** Marca `deleted_at` con la fecha y hora UTC actual. Un libro con préstamos abiertos no puede darse de baja.

---

## 3. Entidades `Loan` y `LoanItem` (T-05)

### 3.1. Invariantes de `Loan`
1. **Límites de Ítems (CA-06.1):** Un préstamo contiene estrictamente entre 1 y 3 libros.
2. **No Duplicados (CA-06.2):** No se permite incluir el mismo libro más de una vez en el mismo préstamo. Si se repite, lanza `DuplicateBookInLoanException` (422).
3. **Estado Calculado (CA-08.3):** El estado nunca se persiste en base de datos; se calcula dinámicamente frente al instante cronológico actual ($now$ en UTC):
   - `closed`: Todas las líneas del préstamo tienen fecha de devolución (`returned_at IS NOT NULL`).
   - `overdue`: Al menos una línea no ha sido devuelta y su fecha de vencimiento es anterior a la actual ($due\_at < now$).
   - `open`: Existen líneas no devueltas y ninguna de ellas se encuentra vencida.

### 3.2. Invariantes de `LoanItem`
1. **Congelamiento Histórico (RN-4 & ADR-004):** En el momento del préstamo se congelan $titulo$, $isbn$ y $nombre\_de\_categoria$. Ediciones posteriores del catálogo no alteran estos registros históricos.
2. **Plazo Contractual (RN-2):** La fecha de vencimiento es exactamente $due\_at = loaned\_at + 14\text{ días}$.
3. **Devolución Única (`returnItem()`):** Registra $returned\_at$. Si la línea ya estaba devuelta, lanza `ItemAlreadyReturnedException` (409).

---

## 4. Entidades `Member` y `User` (T-06)

### 4.1. Entidad `Member`
- Representa a un socio de la biblioteca con $id$, $full\_name$ y $document\_number$.
- Límite de 3 libros sin devolver en total (CA-06.4).
- Bloqueo por mora: Si posee algún libro vencido, se rechaza la solicitud con `MemberHasOverdueException` (409).

### 4.2. Entidad `User`
- Representa a un usuario con acceso al sistema ($id$, $username$, $password\_hash$, $role$).
- **Normalización:** El nombre de usuario se guarda siempre recortado de espacios y en minúsculas (`strtolower(trim($username))`).
- **Roles válidos:** Exclusivamente `admin` o `librarian`.
- **Restricción de rol (CA-02.3):** El rol `admin` no se puede crear por la API.

---

## 5. Jerarquía de Excepciones de Dominio

Todas las excepciones del dominio extienden de `Library\Domain\Exceptions\LibraryBusinessException`:
- `BookNotAvailableException` (Código: `BOOK_NOT_AVAILABLE`, HTTP 409)
- `DuplicateBookInLoanException` (Código: `DUPLICATE_BOOK_IN_LOAN`, HTTP 422)
- `ItemAlreadyReturnedException` (Código: `ITEM_ALREADY_RETURNED`, HTTP 409)
- `MemberHasOverdueException` (Código: `MEMBER_HAS_OVERDUE`, HTTP 409)
- `MemberLimitExceededException` (Código: `MEMBER_LIMIT_EXCEEDED`, HTTP 422)
- `ValidationException` (Código: `VALIDATION_ERROR`, HTTP 422)

---

## 6. Evidencia y Pruebas Unitarias
- Suite: `tests/Unit/Domain/EntitiesTest.php`
- Cobertura: Retiro y reabastecimiento de libros, límites de ejemplares, inmutabilidad de líneas de préstamo, devolución repetida rechazada, estado calculado (`open`, `overdue`, `closed`), y normalización de usuarios.
