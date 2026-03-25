# Guía de Integración y Pruebas
## Examen BD para Negocios Digitales — Boutique E-commerce

---

## Requisitos previos

- MySQL 8.0 o superior
- MySQL Workbench instalado
- Base de datos del docente restaurada (`EXAMEN_180326.sql`)
- Carpeta del proyecto: `C:\Users\ASUS\Desktop\sql\`

---

## PARTE 1 — Integración de cambios

### Paso 1 — Restaurar la base del docente

1. Abrir MySQL Workbench
2. Ir a **Server → Data Import**
3. Seleccionar **Import from Self-Contained File**
4. Buscar `EXAMEN_180326 (1).sql`
5. En **Default Target Schema** escribir: `ecomerce_230426`
6. Hacer clic en **Start Import**
7. Verificar en el panel izquierdo que aparece el esquema `ecommerce`

> Si el esquema ya existe con datos viejos, ejecutar primero:
> ```sql
> DROP SCHEMA IF EXISTS ecommerce;
> CREATE SCHEMA ecommerce DEFAULT CHARACTER SET utf8mb4;
> ```

---

### Paso 2 — Verificar el esquema restaurado

Abrir una nueva pestaña SQL en Workbench y ejecutar:

```sql
USE `ecomerce_230426`;

-- Verificar que las tablas críticas existen
SHOW TABLES;

-- Verificar estructura de las 3 tablas clave
DESCRIBE sesiones;
DESCRIBE carritos;
DESCRIBE carrito_detalle;
```

**Resultado esperado:** deben aparecer al menos 16 tablas incluyendo `sesiones`, `carritos`, `carrito_detalle`, `pedidos`, `transacciones_financieras`, `metodos_pago`, `bitacora`.

---

### Paso 3 — Verificar la tabla bitácora

El archivo `02_triggers.sql` escribe en `bitacora`. Confirmar sus columnas antes de ejecutar los triggers:

```sql
DESCRIBE bitacora;
```

Si las columnas `tabla`, `evento`, `referencia_id`, `descripcion`, `fecha_evento` no existen con esos nombres exactos, ajustar los INSERT dentro de `02_triggers.sql` antes de ejecutarlo.

---

### Paso 4 — Verificar tabla persona_fisica

La vista del Test 07 usa `persona_fisica` para obtener el nombre del comprador. Confirmar:

```sql
DESCRIBE persona_fisica;
-- Buscar columnas: id, nombre, apellido_paterno, apellido_materno
```

Si los nombres de columna difieren, ajustar la vista en `test_07_vista.sql`.

---

### Paso 5 — Ejecutar los archivos en orden

Abrir cada archivo en Workbench (**File → Open SQL Script**) y ejecutar con `Ctrl+Shift+Enter`.

**ORDEN OBLIGATORIO:**

```
① BD/01_funciones.sql
② BD/02_triggers.sql
③ BD/05_indices.sql
④ BD/03_sp_simular.sql
⑤ BD/04_sp_finalizar.sql
```

> No alterar el orden. Las funciones deben existir antes que los SPs.
> Los índices se crean antes de la simulación para que apliquen desde el inicio.

---

### Paso 6 — Verificar que todo se instaló correctamente

```sql
USE `ecomerce_230426`;

-- Funciones (deben aparecer 2)
SHOW FUNCTION STATUS WHERE Db = 'ecommerce';

-- Triggers (deben aparecer 9)
SHOW TRIGGERS;

-- Procedimientos (deben aparecer 2)
SHOW PROCEDURE STATUS WHERE Db = 'ecommerce';

-- Índices en carrito_detalle
SHOW INDEX FROM carrito_detalle;

-- Índices en sesiones
SHOW INDEX FROM sesiones;
```

---

### Paso 7 — Limpieza de datos previos (si hay registros viejos)

Si la BD ya tiene datos en carritos/pedidos, limpiar en este orden por las FKs:

```sql
USE `ecomerce_230426`;

-- 1. Limpiar bitácora asociada a las tablas a depurar
DELETE FROM bitacora
WHERE tabla IN ('carrito_detalle', 'carritos', 'pedidos', 'transacciones_financieras');

-- 2. Detalle del carrito (depende de carritos)
DELETE FROM carrito_detalle;

-- 3. Transacciones (depende de pedidos y metodos_pago)
DELETE FROM transacciones_financieras;

-- 4. Pedidos (depende de carritos)
DELETE FROM pedidos;

-- 5. Carritos (depende de sesiones)
DELETE FROM carritos;

-- 6. Sesiones
DELETE FROM sesiones;

-- Reiniciar auto_increment
ALTER TABLE sesiones               AUTO_INCREMENT = 1;
ALTER TABLE carritos               AUTO_INCREMENT = 1;
ALTER TABLE pedidos                AUTO_INCREMENT = 1;
ALTER TABLE transacciones_financieras AUTO_INCREMENT = 1;
```

---

## PARTE 2 — Pruebas (Tests 01 al 10)

> Antes de cada test, anotar cuántos registros hay con:
> ```sql
> SELECT COUNT(*) FROM sesiones;
> SELECT COUNT(*) FROM carritos;
> SELECT COUNT(*) FROM pedidos;
> ```

---

### TEST 01 — 1 compra completa

**Objetivo:** Verificar el flujo completo: sesión → carrito → detalle → pedido → pago.

```sql
USE `ecomerce_230426`;

-- Paso A: Simular el carrito
CALL sp_simular_carrito(1, NULL, CURDATE());

-- Paso B: Ver el carrito generado
SELECT c.id, c.sesion_id, c.total_productos,
       c.monto_aproximado, c.fecha_creacion,
       c.fecha_actualizacion, c.estatus
FROM carritos c
ORDER BY c.id DESC LIMIT 1;

-- Paso C: Ver el detalle con timestamps crecientes
SELECT cd.carrito_id, p.nombre AS producto,
       cd.fecha_registro, cd.precio_unitario
FROM carrito_detalle cd
JOIN productos p ON p.id = cd.producto_id
WHERE cd.carrito_id = (SELECT MAX(id) FROM carritos)
ORDER BY cd.fecha_registro;

-- Paso D: Ver la sesión (fin provisional asignado)
SELECT s.fecha_registro, s.fecha_inicio, s.fecha_fin,
       fn_duracion_sesion(s.fecha_inicio, s.fecha_fin) AS duracion
FROM sesiones s
ORDER BY s.id DESC LIMIT 1;

-- Paso E: Finalizar la compra (usar IDs reales del paso B y D)
-- Sustituir 1 y 1 por el carrito_id y metodo_pago_id reales
CALL sp_finalizar_compra(
    (SELECT MAX(id) FROM carritos),
    (SELECT MIN(id) FROM metodos_pago WHERE estatus = 'Vigente')
);

-- Paso F: Verificar pedido y pago creados
SELECT p.id, p.estatus, p.importe_total, p.aprobacion FROM pedidos ORDER BY id DESC LIMIT 1;
SELECT tf.id, tf.tipo, tf.origen, tf.monto, tf.estatus FROM transacciones_financieras ORDER BY id DESC LIMIT 1;

-- Paso G: Verificar que fecha_fin de sesión se actualizó (Enfoque C)
SELECT s.fecha_inicio, s.fecha_fin,
       fn_duracion_sesion(s.fecha_inicio, s.fecha_fin) AS duracion_final
FROM sesiones s ORDER BY id DESC LIMIT 1;
```

**Captura de pantalla:** resultado del Paso C (timestamps crecientes) y Paso G (fecha_fin actualizada).

---

### TEST 02 — 10 compras de Perfumería

```sql
USE `ecomerce_230426`;

-- Verificar el categoria_id de Perfumería
SELECT id, nombre FROM categorias WHERE nombre LIKE '%erfum%';

-- Ejecutar con el ID encontrado (ejemplo: 3)
CALL sp_simular_carrito(10, 3, CURDATE());

-- Verificar que los productos son de Perfumería
SELECT cd.carrito_id, p.nombre, cat.nombre AS categoria
FROM carrito_detalle cd
JOIN productos p ON p.id = cd.producto_id
JOIN productos_categorias pc ON pc.producto_id = p.id
JOIN categorias cat ON cat.id = pc.categoria_id
WHERE pc.categoria_id = 3
ORDER BY cd.carrito_id, cd.fecha_registro
LIMIT 30;
```

**Captura:** mostrar los 10 carritos con sus productos de Perfumería.

---

### TEST 03 — 100 compras en 2026

```sql
USE `ecomerce_230426`;

CALL sp_simular_carrito(100, NULL, '2026-06-15');

-- Verificar que las sesiones caen en 2026
SELECT COUNT(*) AS total_sesiones_2026
FROM sesiones
WHERE YEAR(fecha_registro) = 2026;

-- Verificar distribución horaria (debe estar entre 10:00 y 21:00)
SELECT MIN(TIME(fecha_registro)) AS hora_min,
       MAX(TIME(fecha_fin))      AS hora_max
FROM sesiones
WHERE YEAR(fecha_registro) = 2026;
```

**Captura:** COUNT de sesiones en 2026 y rango horario.

---

### TEST 04 — 1000 compras Ropa para mujer

```sql
USE `ecomerce_230426`;

-- Verificar categoria_id de Ropa para mujer
SELECT id, nombre FROM categorias WHERE nombre LIKE '%mujer%';

-- Ejecutar (puede tardar 30–60 segundos)
CALL sp_simular_carrito(1000, 1, CURDATE());  -- ajustar ID de categoría

SELECT COUNT(*) AS total_carritos FROM carritos;
```

---

### TEST 05 — 500 compras Ropa para hombre

```sql
USE `ecomerce_230426`;

SELECT id, nombre FROM categorias WHERE nombre LIKE '%hombre%';

CALL sp_simular_carrito(500, 2, CURDATE());  -- ajustar ID

SELECT COUNT(*) AS total_carritos FROM carritos;
```

---

### TEST 06 — 10 000 compras generales

```sql
USE `ecomerce_230426`;

-- Este test puede tardar 3–8 minutos. Aumentar timeout si es necesario:
SET SESSION wait_timeout = 600;
SET SESSION interactive_timeout = 600;

CALL sp_simular_carrito(10000, NULL, CURDATE());

-- Verificar totales
SELECT
    (SELECT COUNT(*) FROM sesiones)               AS total_sesiones,
    (SELECT COUNT(*) FROM carritos)               AS total_carritos,
    (SELECT COUNT(*) FROM carrito_detalle)        AS total_lineas_detalle;
```

**Captura:** conteos finales de las 3 tablas.

---

### TEST 07 — Vista integral del proceso de compra

```sql
USE `ecomerce_230426`;

-- Ejecutar el archivo de la vista
SOURCE C:/Users/ASUS/Desktop/sql/Tests/test_07_vista.sql;

-- Consultar la vista
SELECT * FROM v_proceso_compra_completo LIMIT 10;

-- Ver columnas disponibles
DESCRIBE v_proceso_compra_completo;
```

**Captura:** primeras 10 filas de la vista con todas las columnas visibles.

---

### TEST 08 — Línea de tiempo por carrito

```sql
USE `ecomerce_230426`;

-- Elegir un carrito con varios productos
SELECT carrito_id, COUNT(*) AS num_productos
FROM carrito_detalle
GROUP BY carrito_id
ORDER BY num_productos DESC
LIMIT 5;

-- Ejecutar la consulta con un carrito_id específico (ej. el de mayor productos)
SET @p_carrito_id = 5;  -- sustituir con el ID real

SOURCE C:/Users/ASUS/Desktop/sql/Tests/test_08_timeline.sql;
```

**Captura:** tabla con columnas `orden_seleccion`, `producto`, `momento_seleccion`, `tiempo_desde_anterior`.
Verificar que `tiempo_desde_anterior` esté entre 00:02:00 y 00:08:00 en cada fila.

---

### TEST 09 — Métricas de tiempo

```sql
USE `ecomerce_230426`;

SOURCE C:/Users/ASUS/Desktop/sql/Tests/test_09_metricas.sql;
```

**Captura:** tabla con `duracion_sesion`, `tiempo_seleccion_productos`, `tiempo_aprobacion_pago`.

---

### TEST 10 — Dashboard de pagos por origen

**En MySQL Workbench — exportar datos:**

```sql
USE `ecomerce_230426`;

-- Consulta base para el dashboard
SELECT
    tf.origen,
    COUNT(*)            AS num_transacciones,
    SUM(tf.monto)       AS monto_total,
    AVG(tf.monto)       AS monto_promedio,
    MIN(tf.fecha_registro) AS primera_transaccion,
    MAX(tf.fecha_registro) AS ultima_transaccion
FROM transacciones_financieras tf
WHERE tf.estatus = 'Aprobado'
GROUP BY tf.origen
ORDER BY monto_total DESC;
```

**Exportar a CSV desde Workbench:**
1. Ejecutar la consulta anterior
2. En el panel de resultados → clic en el ícono de exportar (diskette con flecha)
3. Guardar como `pagos_por_origen.csv` en `/Tests/`

**Importar en Power BI (u otra herramienta):**
1. **Power BI Desktop** → Obtener datos → Archivo CSV
2. Seleccionar `pagos_por_origen.csv`
3. Crear gráfico de barras o pastel con:
   - Eje X / Leyenda: `origen`
   - Valores: `monto_total` o `num_transacciones`
4. Exportar captura del dashboard como imagen PNG

---

## PARTE 3 — Consultas de validación de coherencia temporal

Ejecutar estas consultas después de cualquier simulación. **Deben retornar 0 filas:**

```sql
USE `ecomerce_230426`;

-- V1: Sesiones donde fecha_fin <= fecha_inicio (incoherencia)
SELECT id, fecha_inicio, fecha_fin
FROM sesiones
WHERE fecha_fin IS NOT NULL
  AND fecha_fin <= fecha_inicio;

-- V2: Sesiones donde fecha_inicio = fecha_registro (deben ser distintas)
SELECT id, fecha_registro, fecha_inicio
FROM sesiones
WHERE fecha_inicio = fecha_registro;

-- V3: Carritos con fecha_actualizacion anterior al primer detalle
SELECT c.id AS carrito_id,
       c.fecha_actualizacion,
       MIN(cd.fecha_registro) AS primer_producto
FROM carritos c
JOIN carrito_detalle cd ON cd.carrito_id = c.id
GROUP BY c.id, c.fecha_actualizacion
HAVING c.fecha_actualizacion < MIN(cd.fecha_registro);

-- V4: Carritos con fecha_actualizacion distinta al último detalle
SELECT c.id AS carrito_id,
       c.fecha_actualizacion,
       MAX(cd.fecha_registro) AS ultimo_producto
FROM carritos c
JOIN carrito_detalle cd ON cd.carrito_id = c.id
GROUP BY c.id, c.fecha_actualizacion
HAVING ABS(TIMESTAMPDIFF(SECOND, c.fecha_actualizacion, MAX(cd.fecha_registro))) > 1;

-- V5: Sesiones fuera del horario boutique (antes de 10:00 o después de 21:00)
SELECT id, fecha_registro, TIME(fecha_registro) AS hora
FROM sesiones
WHERE TIME(fecha_registro) < '10:00:00'
   OR TIME(fecha_fin)      > '21:00:00';
```

---

## Resumen rápido de categorías (buscar IDs reales)

```sql
USE `ecomerce_230426`;
SELECT id, nombre FROM categorias ORDER BY id;
```

Usar los IDs reales en las llamadas a `sp_simular_carrito`.

---

## Estructura de archivos del proyecto

```
C:\Users\ASUS\Desktop\sql\
├── EXAMEN_180326 (1).sql          ← BD del docente (no modificar)
├── GUIA_INTEGRACION_Y_PRUEBAS.md  ← este archivo
├── BD/
│   ├── 01_funciones.sql           ← fn_tiempo_realista, fn_duracion_sesion
│   ├── 02_triggers.sql            ← 9 triggers
│   ├── 03_sp_simular.sql          ← sp_simular_carrito
│   ├── 04_sp_finalizar.sql        ← sp_finalizar_compra
│   └── 05_indices.sql             ← 6 índices justificados
├── Tests/
│   ├── test_07_vista.sql          ← Vista integral
│   ├── test_08_timeline.sql       ← Línea de tiempo por carrito
│   └── test_09_metricas.sql       ← Métricas de duración y tiempos
└── docs/
    └── superpowers/specs/
        └── 2026-03-22-sesiones-carritos-tiempo-design.md
```
