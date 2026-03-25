# Spec: Lógica Temporal Sesiones / Carritos / Carrito Detalle
**Fecha:** 2026-03-22
**Proyecto:** Examen BD para Negocios Digitales — Boutique E-commerce
**Alcance:** fn_duracion_sesion, fn_tiempo_realista, trg_carrito_detalle_ai, sp_simular_carrito, sp_finalizar_compra, índices

---

## 1. Contexto y problema

El SQL del docente (`EXAMEN_180326.sql`) define las tablas pero **no contiene** ningún SP, función ni trigger. Los datos actuales presentan:

- `sesiones.fecha_fin` sin poblar o igual a `fecha_inicio`
- `carritos.fecha_actualizacion` que no cambia por ítem agregado
- `carrito_detalle.fecha_registro` sin orden creciente

El objetivo es implementar la capa lógica completa que garantice coherencia temporal en las tres tablas.

---

## 2. Esquema relevante (del docente)

### `sesiones`
| Columna | Tipo | Notas |
|---|---|---|
| `id` | int unsigned PK | AUTO_INCREMENT |
| `usuario_id` | int unsigned FK | → usuarios |
| `origen` | enum | Plataforma / Liga Externa / Campaña Publicitaria |
| `tipo_dispositivo` | enum | PC / Laptop / Smart Phone / Tablet / etc. |
| `sistema_operativo` | varchar(50) | |
| `codigo_pais` | char(2) | |
| `fecha_registro` | datetime | DEFAULT CURRENT_TIMESTAMP |
| `fecha_inicio` | datetime | nullable; CHECK fecha_inicio >= fecha_registro |
| `fecha_fin` | datetime | nullable; se llena en sp_finalizar_compra (Enfoque C) |
| `estatus` | enum | Activa / Finalizada / Cancelada / Suspendida |

### `carritos`
| Columna | Tipo | Notas |
|---|---|---|
| `id` | int unsigned PK | AUTO_INCREMENT |
| `sesion_id` | int unsigned FK | → sesiones |
| `divisa_id` | int unsigned FK | → divisas (default 1) |
| `total_productos` | int unsigned | DEFAULT 0; actualizado por trigger |
| `monto_aproximado` | double unsigned | DEFAULT 0; actualizado por trigger |
| `fecha_creacion` | datetime | DEFAULT CURRENT_TIMESTAMP |
| `fecha_actualizacion` | datetime | nullable; actualizada por trigger en cada ítem |
| `estatus` | enum | Activo / Abandonado / Cancelado / Convertido / Expirado |

### `carrito_detalle`
| Columna | Tipo | Notas |
|---|---|---|
| `carrito_id` | int unsigned PK | FK → carritos |
| `producto_id` | int unsigned PK | FK → productos; PK compuesta, no repite producto |
| `cantidad` | int unsigned | |
| `precio_unitario` | double unsigned | |
| `fecha_registro` | datetime | Momento de selección del producto; creciente por carrito |
| `fecha_actualizacion` | datetime | nullable |
| `estatus` | enum | Activo / Eliminado / Cancelado / Pendiente |

---

## 3. Decisiones de diseño

| Decisión | Opción elegida | Razón |
|---|---|---|
| Cuándo se asigna `fecha_fin` sesión | **Enfoque C**: en `sp_finalizar_compra` al pago exitoso | Coherencia con modelo de negocio; sp_simular_carrito asigna valor provisional |
| `fecha_registro` vs `fecha_inicio` en sesiones | **Desfasadas 1–3 min** | fecha_registro = login sistema; fecha_inicio = inicio navegación real |
| Actualización de `carritos.fecha_actualizacion` | **Trigger AFTER INSERT** en carrito_detalle | Garantía de integridad independiente del origen del INSERT |
| Intervalo entre productos | **Aleatorio 2–8 minutos** via fn_tiempo_realista | Simula lectura de ficha técnica, talla, color en boutique |
| Productos por carrito | **3–6 productos distintos** por carrito | PK compuesta impide repetir; rango realista para boutique |
| Horario boutique | **10:00–21:00**; t0 limitado a 10:00–18:00 | Margen para que todos los timestamps queden dentro de ventana |

---

## 4. Componentes a implementar

### 4.1 `fn_tiempo_realista(p_fecha_base DATETIME) RETURNS DATETIME`

**Propósito:** Sumar un intervalo aleatorio de 120–480 segundos (2–8 min) a una fecha base.

**Lógica:**
```sql
RETURN TIMESTAMPADD(SECOND, FLOOR(RAND() * 361) + 120, p_fecha_base);
```

**Uso:** Llamada en cada iteración del bucle de productos dentro de `sp_simular_carrito`.

---

### 4.2 `fn_duracion_sesion(p_inicio DATETIME, p_fin DATETIME) RETURNS VARCHAR(8)`

**Propósito:** Calcular duración de sesión en formato HH:MM:SS.

**Lógica:**
```sql
RETURN TIME_FORMAT(TIMEDIFF(p_fin, p_inicio), '%H:%i:%s');
```

**Uso:** Tests 07 y 09; Vista integral.

---

### 4.3 `trg_carrito_detalle_ai` — AFTER INSERT en `carrito_detalle`

**Propósito:** Mantener `carritos.fecha_actualizacion`, `total_productos` y `monto_aproximado` sincronizados con cada ítem agregado.

**Lógica:**
```sql
UPDATE carritos
SET
  fecha_actualizacion = NEW.fecha_registro,
  total_productos     = total_productos + NEW.cantidad,
  monto_aproximado    = monto_aproximado + (NEW.cantidad * NEW.precio_unitario)
WHERE id = NEW.carrito_id;
```

**Efecto en ejemplo del enunciado:**
- INSERT detalle producto 1 con fecha_registro = 11:43 → carrito.fecha_actualizacion = 11:43
- INSERT detalle producto 2 con fecha_registro = 11:46 → carrito.fecha_actualizacion = 11:46

**Cuenta como:** Trigger #1 de los 9 requeridos.

---

### 4.4 `sp_simular_carrito(p_num_compras INT, p_categoria_id INT, p_fecha_base DATE)`

**Parámetros:**
- `p_num_compras`: cantidad de compras a generar (1–10000)
- `p_categoria_id`: NULL = todas las categorías; valor específico = filtra
- `p_fecha_base`: fecha del día a simular (ej. '2026-06-15')

**Flujo por iteración:**

```
1. Elegir @usuario_id aleatorio de usuarios activos

2. Calcular @t0 (fecha_registro sesión):
   hora_base = FLOOR(RAND() * 480) + 600  -- minutos desde medianoche: 600=10:00, 1080=18:00
   @t0 = TIMESTAMP(p_fecha_base) + INTERVAL hora_base MINUTE

3. @t1 (fecha_inicio) = @t0 + RAND(1–3 min)
   -- Cumple CHECK: fecha_inicio >= fecha_registro

4. INSERT sesiones (fecha_registro=@t0, fecha_inicio=@t1, fecha_fin=NULL, estatus='Activa')
   → @sesion_id = LAST_INSERT_ID()

5. @t2 (fecha_creacion carrito) = @t1 + RAND(0–2 min)

6. INSERT carritos (sesion_id=@sesion_id, fecha_creacion=@t2, estatus='Activo')
   → @carrito_id = LAST_INSERT_ID()

7. Elegir @num_productos = FLOOR(RAND()*4) + 3  -- entre 3 y 6

8. Seleccionar @num_productos productos distintos del catálogo
   (con filtro categoria_id si p_categoria_id IS NOT NULL)
   → insertar en tabla temporal tmp_productos_seleccionados

9. SET @t_actual = @t2
   LOOP sobre tmp_productos_seleccionados:
     SET @t_actual = fn_tiempo_realista(@t_actual)
     INSERT carrito_detalle (carrito_id, producto_id, cantidad, precio_unitario, fecha_registro=@t_actual)
     -- trigger trg_carrito_detalle_ai dispara automáticamente

10. @tf (fecha_fin sesión) = fn_tiempo_realista(@t_actual)  -- +2–8 min después del último producto
    -- Validar: si @tf > 21:00, recalcular @t0 más temprano
    UPDATE sesiones SET fecha_fin=@tf, estatus='Finalizada' WHERE id=@sesion_id

FIN LOOP
```

**Garantías:**
- `fecha_fin > fecha_inicio > fecha_registro` ✓
- `carrito.fecha_actualizacion` = timestamp del último producto ✓
- Timestamps en detalle estrictamente crecientes ✓
- Todos los timestamps dentro de ventana 10:00–21:00 ✓
- Productos reales del catálogo, sin repetir en mismo carrito ✓

---

### 4.5 `sp_finalizar_compra(p_carrito_id INT, p_metodo_pago_id INT)`

**Propósito:** Transformar carrito activo en compra completada con integridad transaccional.

**Flujo:**
```
START TRANSACTION;

  1. Validar carrito: EXISTS y estatus = 'Activo'          → ROLLBACK si falla
  2. Validar líneas: COUNT(*) > 0 en carrito_detalle        → ROLLBACK si falla
  3. INSERT pedidos (sesion_id, monto, estatus='Pendiente', fecha_pedido=NOW())
     → @pedido_id; dispara triggers de pedidos
  4. INSERT transacciones_financieras (pedido_id, metodo_pago_id, monto, fecha=NOW(), estatus='Aprobada')
     → dispara triggers de transacciones_financieras
  5. UPDATE carritos SET estatus='Convertido' WHERE id = p_carrito_id
  6. [ENFOQUE C] UPDATE sesiones
     SET fecha_fin = NOW(), estatus = 'Finalizada'
     WHERE id = (SELECT sesion_id FROM carritos WHERE id = p_carrito_id)

COMMIT;
```

---

## 5. Índices

| Nombre | Tabla | Columna(s) | Test justificante |
|---|---|---|---|
| `idx_sesiones_usuario` | sesiones | `usuario_id` | 07, 09 |
| `idx_sesiones_fechas` | sesiones | `fecha_inicio, fecha_fin` | 07, 08, 09 |
| `idx_carritos_sesion` | carritos | `sesion_id` | 07, 08 |
| `idx_detalle_carrito_fecha` | carrito_detalle | `carrito_id, fecha_registro` | 08, 09 |
| `idx_pedidos_fecha` | pedidos | `fecha_pedido` | 03, 07 |
| `idx_prod_cat` | productos_categorias | `categoria_id` | 02, 04, 05 |

---

## 6. Reglas de validación

Estas consultas deben retornar 0 filas después de la simulación:

```sql
-- fecha_fin <= fecha_inicio en sesiones
SELECT id FROM sesiones WHERE fecha_fin <= fecha_inicio;

-- fecha_inicio = fecha_registro en sesiones (deben ser distintas)
SELECT id FROM sesiones WHERE fecha_inicio = fecha_registro;

-- carrito.fecha_actualizacion anterior al primer detalle
SELECT c.id FROM carritos c
JOIN carrito_detalle cd ON cd.carrito_id = c.id
GROUP BY c.id
HAVING c.fecha_actualizacion < MIN(cd.fecha_registro);

-- timestamps no crecientes en detalle (requiere columna de orden)
-- verificar manualmente con Test 08
```

---

## 7. Orden de implementación

```
1. fn_tiempo_realista
2. fn_duracion_sesion
3. trg_carrito_detalle_ai
4. sp_simular_carrito
5. sp_finalizar_compra
6. Índices
7. Tests 01–06 (ejecutar SP con parámetros de cada test)
8. Tests 07–09 (consultas y vista con fn_duracion_sesion)
9. Test 10 (dashboard Power BI / herramienta libre)
```

---

## 8. Archivos a generar

| Archivo | Contenido |
|---|---|
| `/BD/01_funciones.sql` | fn_tiempo_realista, fn_duracion_sesion |
| `/BD/02_triggers.sql` | trg_carrito_detalle_ai + 8 triggers restantes |
| `/BD/03_sp_simular.sql` | sp_simular_carrito |
| `/BD/04_sp_finalizar.sql` | sp_finalizar_compra |
| `/BD/05_indices.sql` | 6 índices justificados |
| `/BD/06_tests.sql` | Llamadas a SP para Tests 01–06 |
| `/Tests/test_07_vista.sql` | Vista integral |
| `/Tests/test_08_timeline.sql` | Línea de tiempo por carrito_id |
| `/Tests/test_09_metricas.sql` | Métricas de duración y tiempos |
