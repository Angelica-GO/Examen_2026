# Log de Asistencia con Inteligencia Artificial
## Examen Práctico — Base de Datos para Negocios Digitales

**Herramienta utilizada:** Claude Code (Anthropic) — Modelo: Claude Sonnet 4.6
**Fecha de sesión:** 22 de marzo de 2026
**Proyecto:** Boutique E-commerce — Lógica temporal sesiones/carritos/detalle

---

## Índice de prompts

| # | Tema | Tipo de asistencia |
|---|---|---|
| 01 | Revisión del plan del examen y análisis del SQL del docente | Análisis |
| 02 | Diseño de lógica temporal para sesiones, carritos y detalle | Diseño |
| 03 | Decisión sobre enfoque de fin_sesion (Enfoque C) | Decisión de arquitectura |
| 04 | Parámetros de fn_tiempo_realista (intervalo boutique) | Diseño de función |
| 05 | Número de productos por carrito y restricción de PK | Diseño de SP |
| 06 | Manejo de fecha_registro vs fecha_inicio en sesiones | Diseño de esquema |
| 07 | Propuesta y elección de enfoque de implementación | Arquitectura |
| 08 | Generación de fn_tiempo_realista y fn_duracion_sesion | Código SQL |
| 09 | Generación de 9 triggers con distribución por tabla | Código SQL |
| 10 | Generación de sp_simular_carrito con loop temporal | Código SQL |
| 11 | Generación de sp_finalizar_compra con TRANSACTION | Código SQL |
| 12 | Generación de índices justificados por test | Código SQL |
| 13 | Generación de consultas Tests 07, 08, 09 | Código SQL |
| 14 | Documentación de integración y pruebas | Documentación |
| 15 | Revisión de entregables vs plan del examen | Revisión |
| 16 | Corrección de bugs: esquema, bitácora, FK faltante | Depuración |

---

## Detalle de prompts y respuestas

---

### Prompt 01 — Revisión del plan y SQL del docente

**Prompt enviado:**
> "usa /brainstorming y revisa el plan que tenemos ya @examen_bd_e-commerce_8f857522.plan.md y @EXAMEN_180326 (1).sql PARA PODER HACER ESTO: Corregir la lógica de la fecha de fin y simular las sesiones para que la fecha de registro/inicio sea diferente a la fecha de fin. La fecha de actualización debe cambiar cada vez que se agregue un producto al carrito. Por ejemplo, si se agrega el 'Producto X' a las 11:43 y a los tres minutos se agrega otro, la actualización debe marcar las 11:46."

**Respuesta / Acción de la IA:**
- Leyó el plan del examen completo (10 actividades, 10 todos pendientes)
- Extrajo del SQL del docente las definiciones exactas de `sesiones`, `carritos` y `carrito_detalle`
- Identificó que el esquema real se llama `ecomerce_230426`
- Confirmó que no existían SP, funciones ni triggers en el archivo del docente
- Estructura de `sesiones`: `fecha_registro`, `fecha_inicio` (con CHECK `>= fecha_registro`), `fecha_fin` nullable
- Estructura de `carritos`: `fecha_creacion`, `fecha_actualizacion` nullable
- Estructura de `carrito_detalle`: `fecha_registro` (momento de selección), PK compuesta `(carrito_id, producto_id)`

**Valor aportado:** Identificación del esquema real antes de diseñar, evitando escribir código sobre suposiciones incorrectas.

---

### Prompt 02 — Decisión sobre fin_sesion (Enfoque C)

**Prompt enviado:**
> Pregunta de la IA: ¿Cuándo se asigna `fecha_fin` de sesión? Opciones A (provisional en SP simular), B (solo en SP finalizar), C (SP simular asigna provisional, SP finalizar sobreescribe al pago).

**Respuesta del usuario:** A

**Acción de la IA:**
- Confirmó Enfoque C híbrido: `sp_simular_carrito` asigna `fecha_fin` provisional al final del loop; `sp_finalizar_compra` la sobreescribe con el timestamp real del pago exitoso.

**Justificación técnica:** Permite consultar `fn_duracion_sesion` en cualquier momento sin obtener NULL, y garantiza coherencia con el pago real en producción.

---

### Prompt 03 — Intervalo de tiempo entre productos

**Prompt enviado:**
> Pregunta de la IA: ¿Qué rango de tiempo entre productos? A) Fijo 3 min, B) Aleatorio 1-5 min, C) Aleatorio 2-8 min (simula ficha técnica, talla, color).

**Respuesta del usuario:** C

**Acción de la IA:**
- Definió `fn_tiempo_realista` con fórmula: `TIMESTAMPADD(SECOND, FLOOR(RAND() * 361) + 120, p_fecha_base)`
- Rango: 120 a 480 segundos (2 a 8 minutos)
- Horario boutique confirmado: 10:00–21:00; inicio limitado a 10:00–18:00 para dejar margen

---

### Prompt 04 — Productos por carrito

**Prompt enviado:**
> Pregunta de la IA: ¿Cuántos productos por carrito? A) 2-3, B) 3-6, C) Parametrizable.

**Respuesta del usuario:** B

**Acción de la IA:**
- Estableció `FLOOR(RAND() * 4) + 3` → entre 3 y 6 productos distintos
- Señaló restricción de PK compuesta `(carrito_id, producto_id)`: no se puede repetir producto en el mismo carrito
- Estrategia: tabla temporal `tmp_prods_sim` con 6 productos distintos por RAND(), usar solo los primeros `v_num_productos`

---

### Prompt 05 — Manejo de fecha_registro vs fecha_inicio

**Prompt enviado:**
> Pregunta de la IA: ¿Cómo manejar `fecha_registro` y `fecha_inicio` de sesión? A) Iguales, B) Desfasadas 1-3 min.

**Respuesta del usuario:** B

**Acción de la IA:**
- `fecha_registro` = momento en que el sistema crea la sesión (login)
- `fecha_inicio` = `fecha_registro` + RAND(1-3 min) = cuando el usuario empieza a navegar
- Cumple el CHECK constraint del docente: `fecha_inicio >= fecha_registro`

---

### Prompt 06 — Elección de enfoque de implementación

**Prompt enviado:**
> Pregunta de la IA: ¿Cómo actualizar `carritos.fecha_actualizacion`? Opción 1 (solo en SP), Opción 2 (SP + trigger AFTER INSERT), Opción 3 (cursor).

**Respuesta del usuario:** Sí (Opción 2)

**Acción de la IA:**
- Diseñó `trg_carrito_detalle_ai`: trigger AFTER INSERT en `carrito_detalle` que actualiza `fecha_actualizacion`, `total_productos` y `monto_aproximado` del carrito
- Ventaja: garantía de integridad independiente del origen del INSERT
- Cuenta como trigger #1 de los 9 requeridos

---

### Prompt 07 — Generación de funciones SQL

**Prompt enviado:**
> Aprobación del diseño completo → generar código.

**Código generado — `01_funciones.sql`:**

```sql
-- fn_tiempo_realista: suma 2-8 minutos aleatorios
CREATE FUNCTION fn_tiempo_realista(p_fecha_base DATETIME)
RETURNS DATETIME DETERMINISTIC
BEGIN
    RETURN TIMESTAMPADD(SECOND, FLOOR(RAND() * 361) + 120, p_fecha_base);
END

-- fn_duracion_sesion: devuelve HH:MM:SS
CREATE FUNCTION fn_duracion_sesion(p_inicio DATETIME, p_fin DATETIME)
RETURNS VARCHAR(8) DETERMINISTIC
BEGIN
    IF p_inicio IS NULL OR p_fin IS NULL THEN RETURN '00:00:00'; END IF;
    IF p_fin <= p_inicio THEN RETURN '00:00:00'; END IF;
    RETURN TIME_FORMAT(TIMEDIFF(p_fin, p_inicio), '%H:%i:%s');
END
```

---

### Prompt 08 — Generación de 9 triggers

**Prompt enviado:**
> Continuar con la implementación.

**Código generado — `02_triggers.sql`:**

Distribución de los 9 triggers:

| # | Tabla | Tipo | Propósito |
|---|---|---|---|
| 1 | `carrito_detalle` | AFTER INSERT | Actualiza `fecha_actualizacion`, `total_productos`, `monto_aproximado` del carrito |
| 2 | `pedidos` | AFTER INSERT | Registra creación en bitácora |
| 3 | `pedidos` | BEFORE UPDATE | Valida transición de estatus; actualiza `fecha_actualizacion` |
| 4 | `pedidos` | AFTER UPDATE | Registra cambio de estatus en bitácora |
| 5 | `metodos_pago` | BEFORE INSERT | Valida 16 dígitos y formato MM/YY |
| 6 | `metodos_pago` | AFTER UPDATE | Auditoría en bitácora |
| 7 | `metodos_pago` | AFTER DELETE | Auditoría en bitácora |
| 8 | `transacciones_financieras` | BEFORE INSERT | Valida monto > 0 y método vigente |
| 9 | `transacciones_financieras` | AFTER INSERT | Registra ingreso en bitácora |

---

### Prompt 09 — Generación de sp_simular_carrito

**Prompt enviado:**
> Continuar con la implementación.

**Código generado — `03_sp_simular.sql`:**

Parámetros: `p_num_compras INT`, `p_categoria_id INT UNSIGNED`, `p_fecha_base DATE`

Flujo por iteración:
1. Elegir `usuario_id` aleatorio con estatus 'Activo'
2. Calcular `t0` (fecha_registro): RAND dentro de 10:00–18:00
3. Calcular `t1` (fecha_inicio): `t0` + RAND(1–3 min)
4. INSERT sesión con `fecha_fin = NULL`, estatus 'Activa'
5. Calcular `t2` (fecha_creacion carrito): `t1` + RAND(0–2 min)
6. INSERT carrito vinculado a sesión
7. Seleccionar 3–6 productos distintos via tabla temporal + RAND()
8. Loop de productos: `t_actual = fn_tiempo_realista(t_anterior)` → INSERT detalle → trigger actualiza carrito
9. `tf = fn_tiempo_realista(t_actual)` → UPDATE sesión con `fecha_fin` y estatus 'Finalizada'

---

### Prompt 10 — Generación de sp_finalizar_compra

**Prompt enviado:**
> Continuar con la implementación.

**Código generado — `04_sp_finalizar.sql`:**

Parámetros: `p_carrito_id INT UNSIGNED`, `p_metodo_pago_id INT UNSIGNED`

Flujo con START TRANSACTION / COMMIT / ROLLBACK:
1. Validar carrito existe y estatus = 'Activo' → ROLLBACK si falla
2. Validar que tenga líneas activas → ROLLBACK si falla
3. Validar método de pago vigente → ROLLBACK si falla
4. INSERT pedido (estatus 'Pendiente', aprobacion=0)
5. INSERT transacción financiera (tipo='Compra', estatus='Aprobado')
6. UPDATE pedido a estatus='Pagado', aprobacion=1
7. UPDATE carrito a estatus='Convertido'
8. **Enfoque C:** UPDATE sesión: `fecha_fin = NOW()`, estatus='Finalizada'
9. COMMIT

---

### Prompt 11 — Generación de índices y consultas de tests

**Código generado — `05_indices.sql`:**

| Índice | Tabla | Columnas | Test |
|---|---|---|---|
| `idx_sesiones_usuario` | sesiones | `usuario_id` | 07, 09 |
| `idx_sesiones_fechas` | sesiones | `fecha_inicio, fecha_fin` | 07, 08, 09 |
| `idx_detalle_carrito_fecha` | carrito_detalle | `carrito_id, fecha_registro` | 08, 09 |
| `idx_pedidos_fecha` | pedidos | `fecha_registro` | 03, 07 |
| `idx_prod_cat` | productos_categorias | `categoria_id` | 02, 04, 05 |

**Código generado — Tests 07, 08, 09:**
- `test_07_vista.sql`: Vista `v_proceso_compra_completo` con JOIN sesión→carrito→pedido→pago
- `test_08_timeline.sql`: ROW_NUMBER + LAG para orden cronológico de productos
- `test_09_metricas.sql`: MIN/MAX de `fecha_registro` en detalle para calcular tiempo de selección

---

### Prompt 12 — Revisión de entregables y corrección de bugs

**Prompt enviado:**
> "revisa @examen_bd_e-commerce_8f857522.plan.md y revisa si todas las actividades y entregables sin cumplieron"

**Bugs detectados por la IA al comparar con el SQL real del docente:**

| Bug | Causa raíz | Corrección |
|---|---|---|
| `USE ecommerce` en 7 archivos | BD real se llama `ecomerce_230426` | Reemplazado en todos los archivos |
| `evento` en INSERT bitácora | La columna real es `operacion` (enum 'Insert'/'Update'/'Delete') | Corregido |
| `referencia_id` en INSERT bitácora | Columna no existe en el esquema real | Eliminada; info movida a `descripcion` |
| `fecha_evento` en INSERT bitácora | La columna real es `fecha_hora` (con DEFAULT CURRENT_TIMESTAMP) | Corregido |
| Faltaba columna `usuario` en bitácora | `usuario varchar(100) NOT NULL` es obligatoria | Agregado `USER()` en todos los triggers |
| UPDATE pedido en trigger 9 | No hay FK directa transaccion→pedido en el esquema | Movido a `sp_finalizar_compra` |

---

## Resumen de archivos generados con asistencia de IA

```
BD/
├── 01_funciones.sql     ← fn_tiempo_realista, fn_duracion_sesion
├── 02_triggers.sql      ← 9 triggers (carrito_detalle, pedidos, metodos_pago, transacciones)
├── 03_sp_simular.sql    ← sp_simular_carrito (loop con lógica temporal completa)
├── 04_sp_finalizar.sql  ← sp_finalizar_compra (TRANSACTION + Enfoque C)
└── 05_indices.sql       ← 6 índices justificados técnicamente

Tests/
├── test_07_vista.sql    ← Vista v_proceso_compra_completo
├── test_08_timeline.sql ← Línea de tiempo por carrito_id
└── test_09_metricas.sql ← Métricas de duración y tiempos

GUIA_INTEGRACION_Y_PRUEBAS.md ← Instrucciones paso a paso para Tests 01–10
docs/superpowers/specs/2026-03-22-sesiones-carritos-tiempo-design.md ← Spec de diseño
```

---

## Declaración de uso de IA

El equipo utilizó inteligencia artificial (Claude Code / Claude Sonnet 4.6 de Anthropic) como herramienta de apoyo para:

- **Análisis del esquema** proporcionado por el docente
- **Diseño de la lógica temporal** de sesiones, carritos y carrito_detalle
- **Generación de código SQL** (funciones, triggers, procedimientos, índices, vistas)
- **Revisión y depuración** de bugs (columnas incorrectas de bitácora, nombre del esquema)
- **Documentación** de integración y pruebas

Todo el código fue **revisado por el equipo** antes de su ejecución. Las decisiones de diseño (Enfoque C, intervalos de tiempo, distribución de triggers) fueron tomadas por el equipo y ejecutadas con apoyo de la IA.

---

*Generado el 22 de marzo de 2026 — Examen Práctico BD para Negocios Digitales*
