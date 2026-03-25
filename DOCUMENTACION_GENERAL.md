# Documentación General del Proyecto
## Boutique E-commerce — Base de Datos para Negocios Digitales

---

> **¿Para quién es esta guía?**
> Para cualquier integrante del equipo que necesite entender qué se hizo, cómo funciona y cómo ejecutarlo, aunque no tenga mucha experiencia con bases de datos.

---

## ¿Qué es este proyecto?

Tenemos una base de datos de una **tienda en línea tipo boutique** (ropa y perfumería). El objetivo del examen fue **corregir y mejorar** esa base de datos para que:

1. Registre correctamente **cuándo entra y cuándo sale** un usuario de la tienda
2. Lleve un historial de **qué productos agregó al carrito y a qué hora**
3. Pueda **procesar pagos** de forma segura
4. Genere **estadísticas reales** de las compras

Todo esto se hace con código SQL que se ejecuta en **MySQL Workbench**.

---

## ¿Qué archivos tenemos?

```
C:\Users\ASUS\Desktop\sql\
│
├── EXAMEN_180326 (1).sql          ← Base de datos original del docente (NO modificar)
├── DOCUMENTACION_GENERAL.md       ← Este archivo
├── GUIA_INTEGRACION_Y_PRUEBAS.md  ← Guía técnica paso a paso
│
├── BD\
│   ├── ecommerce_completo.sql     ← ⭐ ARCHIVO PRINCIPAL — todo el código en uno
│   ├── 01_funciones.sql           ← Solo las funciones (parte del archivo principal)
│   ├── 02_triggers.sql            ← Solo los triggers (parte del archivo principal)
│   ├── 03_sp_simular.sql          ← Solo el SP de simulación
│   ├── 04_sp_finalizar.sql        ← Solo el SP de pago
│   ├── 05_indices.sql             ← Solo los índices
│   └── diccionario_datos.md       ← Explicación de cada tabla y columna
│
├── Tests\
│   ├── test_07_vista.sql          ← Consulta del Test 07
│   ├── test_08_timeline.sql       ← Consulta del Test 08
│   └── test_09_metricas.sql       ← Consulta del Test 09
│
└── Asistencia_IA\
    └── log_prompts_IA.md          ← Registro de uso de IA (convertir a PDF)
```

> **Regla de oro:** El archivo que necesitas ejecutar para todo es **`BD/ecommerce_completo.sql`**. Los demás archivos en `/BD` son las piezas por separado, por si necesitas revisar algo específico.

---

## Paso 1 — Abrir MySQL Workbench

MySQL Workbench es el programa donde vamos a trabajar. Tiene un aspecto similar a este:

```
┌─────────────────────────────────────────┐
│  MySQL Workbench                        │
│  [Conexión local] → hacer doble clic   │
└─────────────────────────────────────────┘
```

1. Abre **MySQL Workbench** desde el menú de inicio
2. Haz doble clic en la conexión que diga **"Local instance"** o **"localhost"**
3. Si te pide contraseña, ingresa la de tu instalación de MySQL (normalmente `root`)
4. Verás el editor de SQL en el centro de la pantalla

---

## Paso 2 — Cargar la base de datos del docente

Esto solo se hace **una vez** al inicio.

1. En Workbench, ve al menú: **Server → Data Import**
2. Selecciona: **"Import from Self-Contained File"**
3. Haz clic en los tres puntos `...` y busca el archivo:
   ```
   C:\Users\ASUS\Desktop\sql\EXAMEN_180326 (1).sql
   ```
4. En el campo **"Default Target Schema"** escribe: `ecomerce_230426`
5. Haz clic en **"Start Import"**
6. Espera a que termine (puede tardar 1–2 minutos)
7. Cuando diga **"Import completed"**, cierra esa ventana

**¿Cómo saber si funcionó?**
En el panel izquierdo deberías ver `ecomerce_230426` en la lista de esquemas. Si no aparece, haz clic en el botón de actualizar (ícono de flecha circular).

---

## Paso 3 — Ejecutar el archivo principal

Este paso instala todas las mejoras que hicimos: funciones, triggers, índices y procedimientos.

1. En Workbench, ve al menú: **File → Open SQL Script**
2. Busca y abre el archivo:
   ```
   C:\Users\ASUS\Desktop\sql\BD\ecommerce_completo.sql
   ```
3. El archivo se abrirá en el editor de Workbench
4. Presiona **`Ctrl + Shift + Enter`** para ejecutar TODO el archivo
5. Espera a que termine

**¿Cómo saber si funcionó?**
En la parte inferior de Workbench verás el panel de resultados. Si todo está bien, verás mensajes en verde. Si hay errores, aparecerán en rojo.

> ⚠️ **Importante:** Si ves un error que dice algo como "Table doesn't exist" o "Unknown column", significa que alguna tabla del docente tiene un nombre diferente. Revisa la sección de **Solución de problemas** al final de este documento.

---

## Paso 4 — Verificar que todo se instaló bien

Abre una nueva pestaña de SQL (clic en el ícono `+` junto a las pestañas) y ejecuta estas consultas **una por una**:

```sql
USE `ecomerce_230426`;

-- ¿Cuántas funciones instalamos? Debe mostrar 2
SHOW FUNCTION STATUS WHERE Db = 'ecomerce_230426';

-- ¿Cuántos triggers instalamos? Debe mostrar 9
SHOW TRIGGERS;

-- ¿Cuántos procedimientos instalamos? Debe mostrar 2
SHOW PROCEDURE STATUS WHERE Db = 'ecomerce_230426';
```

Si ves 2 funciones, 9 triggers y 2 procedimientos — **¡todo está listo!**

---

## ¿Qué instalamos exactamente?

### Funciones (2)

Las funciones son como calculadoras que usamos en nuestras consultas.

#### `fn_tiempo_realista`
- **¿Qué hace?** Agrega entre 2 y 8 minutos aleatorios a una hora dada
- **¿Para qué sirve?** Para simular que una persona tarda un rato en elegir cada producto en la boutique (leer la etiqueta, ver la talla, el color, etc.)
- **Ejemplo:**
  ```sql
  SELECT fn_tiempo_realista('2026-06-15 11:43:00');
  -- Resultado: algo entre 11:45:00 y 11:51:00
  ```

#### `fn_duracion_sesion`
- **¿Qué hace?** Calcula cuánto tiempo duró algo, dado un inicio y un fin
- **¿Para qué sirve?** Para saber cuánto tiempo estuvo el usuario en la tienda, cuánto tardó en elegir productos, etc.
- **Ejemplo:**
  ```sql
  SELECT fn_duracion_sesion('2026-06-15 10:05:00', '2026-06-15 10:52:23');
  -- Resultado: '00:47:23'  (47 minutos y 23 segundos)
  ```

---

### Triggers (9)

Un trigger es código que **se ejecuta automáticamente** cuando alguien inserta, modifica o elimina un registro. No hay que llamarlos manualmente — se activan solos.

| # | Nombre | Se activa cuando... | ¿Qué hace? |
|---|---|---|---|
| 1 | `trg_carrito_detalle_ai` | Se agrega un producto al carrito | Actualiza la hora del carrito y suma el monto |
| 2 | `trg_pedidos_ai` | Se crea un pedido | Registra en el historial (bitácora) |
| 3 | `trg_pedidos_bu` | Se quiere cambiar el estatus de un pedido | Verifica que el cambio sea válido |
| 4 | `trg_pedidos_au` | Cambia el estatus de un pedido | Registra el cambio en el historial |
| 5 | `trg_metodos_pago_bi` | Se agrega una tarjeta de pago | Verifica que tenga 16 dígitos y formato válido |
| 6 | `trg_metodos_pago_au` | Se modifica una tarjeta | Registra el cambio en el historial |
| 7 | `trg_metodos_pago_ad` | Se elimina una tarjeta | Registra la eliminación en el historial |
| 8 | `trg_trans_bi` | Se va a registrar un pago | Verifica que el monto sea mayor a $0 y que la tarjeta esté vigente |
| 9 | `trg_trans_ai` | Se registra un pago | Anota el ingreso en el historial |

> **En español simple:** Los triggers cuidan que nadie ingrese datos incorrectos y llevan un registro automático de todo lo que pasa.

---

### Procedimientos almacenados (2)

Un procedimiento es como una receta de cocina: le dices qué ingredientes usar y él hace todo el trabajo.

#### `sp_simular_carrito` — El simulador de compras

Genera compras de prueba con datos realistas.

**¿Cómo se usa?**
```sql
CALL sp_simular_carrito(número_de_compras, id_categoría, fecha);
```

**Ejemplos:**
```sql
-- Simular 1 compra de cualquier categoría hoy
CALL sp_simular_carrito(1, NULL, CURDATE());

-- Simular 10 compras de Perfumería
CALL sp_simular_carrito(10, 3, CURDATE());

-- Simular 100 compras con fecha en 2026
CALL sp_simular_carrito(100, NULL, '2026-06-15');
```

**¿Qué hace internamente?**
1. Elige un usuario real de la base de datos
2. Crea una sesión con hora de inicio entre las 10:00 y las 18:00
3. Crea un carrito vinculado a esa sesión
4. Agrega entre 3 y 6 productos reales del catálogo
5. Cada producto se agrega con 2–8 minutos de diferencia (simula navegar la tienda)
6. Al final, cierra la sesión con una hora de fin coherente

#### `sp_finalizar_compra` — El procesador de pagos

Convierte un carrito en una compra real. Es el más importante porque usa **transacciones** (si algo falla a mitad, cancela todo para no dejar datos a medias).

**¿Cómo se usa?**
```sql
CALL sp_finalizar_compra(id_del_carrito, id_del_metodo_de_pago);
```

**Ejemplo:**
```sql
-- Finalizar el carrito número 1 con la tarjeta número 1
CALL sp_finalizar_compra(1, 1);
```

**¿Qué hace internamente?**
1. Verifica que el carrito exista y esté activo
2. Verifica que tenga productos
3. Verifica que la tarjeta de pago esté vigente
4. Crea el pedido formal
5. Registra la transacción de pago
6. Marca el carrito como "Convertido"
7. Actualiza la hora de fin de sesión con el momento exacto del pago
8. Si algo falla → cancela todo (ROLLBACK) y muestra un mensaje de error

---

### Índices (5)

Los índices son como el índice de un libro: permiten encontrar información mucho más rápido sin leer todo.

| Índice | Para qué sirve |
|---|---|
| `idx_sesiones_usuario` | Encontrar rápido las sesiones de un usuario |
| `idx_sesiones_fechas` | Calcular duraciones y buscar por rango de fechas |
| `idx_detalle_carrito_fecha` | Ver el orden cronológico de productos por carrito |
| `idx_pedidos_fecha` | Filtrar pedidos por año (útil para Test 03) |
| `idx_prod_cat` | Encontrar rápido los productos de una categoría |

---

## Paso 5 — Ejecutar los Tests

Antes de ejecutar los tests, necesitas saber los IDs de las categorías:

```sql
USE `ecomerce_230426`;
SELECT id, nombre FROM categorias ORDER BY id;
```

Anota los IDs de **Perfumería**, **Ropa para mujer** y **Ropa para hombre**.

---

### TEST 01 — 1 compra completa

```sql
USE `ecomerce_230426`;

-- Generar la compra
CALL sp_simular_carrito(1, NULL, CURDATE());

-- Ver el carrito que se creó
SELECT id, total_productos, monto_aproximado,
       fecha_creacion, fecha_actualizacion, estatus
FROM carritos ORDER BY id DESC LIMIT 1;

-- Ver los productos en orden de tiempo (deben tener horas crecientes)
SELECT cd.fecha_registro, p.nombre, cd.precio_unitario
FROM carrito_detalle cd
JOIN productos p ON p.id = cd.producto_id
WHERE cd.carrito_id = (SELECT MAX(id) FROM carritos)
ORDER BY cd.fecha_registro;

-- Ver la sesión creada
SELECT fecha_registro, fecha_inicio, fecha_fin,
       fn_duracion_sesion(fecha_inicio, fecha_fin) AS duracion
FROM sesiones ORDER BY id DESC LIMIT 1;

-- Finalizar la compra (pagar)
CALL sp_finalizar_compra(
    (SELECT MAX(id) FROM carritos),
    (SELECT MIN(id) FROM metodos_pago WHERE estatus = 'Vigente')
);

-- Verificar pedido y pago
SELECT id, estatus, importe_total, aprobacion
FROM pedidos ORDER BY id DESC LIMIT 1;
```

📸 **Toma captura de pantalla de los resultados**

---

### TEST 02 — 10 compras de Perfumería

```sql
-- Reemplaza el 3 con el ID real de la categoría Perfumería
CALL sp_simular_carrito(10, 3, CURDATE());

SELECT COUNT(*) AS total_carritos FROM carritos;
```

📸 **Toma captura**

---

### TEST 03 — 100 compras en 2026

```sql
CALL sp_simular_carrito(100, NULL, '2026-06-15');

-- Verificar que estén en 2026
SELECT COUNT(*) AS sesiones_en_2026
FROM sesiones WHERE YEAR(fecha_registro) = 2026;
```

📸 **Toma captura**

---

### TEST 04 — 1000 compras Ropa para mujer

```sql
-- Reemplaza el 1 con el ID real de Ropa para mujer
CALL sp_simular_carrito(1000, 1, CURDATE());
SELECT COUNT(*) AS total_carritos FROM carritos;
```

📸 **Toma captura** *(puede tardar 1–2 minutos)*

---

### TEST 05 — 500 compras Ropa para hombre

```sql
-- Reemplaza el 2 con el ID real de Ropa para hombre
CALL sp_simular_carrito(500, 2, CURDATE());
SELECT COUNT(*) AS total_carritos FROM carritos;
```

📸 **Toma captura**

---

### TEST 06 — 10,000 compras generales

```sql
-- Aumentar tiempo de espera antes de ejecutar
SET SESSION wait_timeout = 600;

CALL sp_simular_carrito(10000, NULL, CURDATE());

SELECT
    (SELECT COUNT(*) FROM sesiones)        AS total_sesiones,
    (SELECT COUNT(*) FROM carritos)        AS total_carritos,
    (SELECT COUNT(*) FROM carrito_detalle) AS total_lineas;
```

📸 **Toma captura** *(puede tardar 5–10 minutos)*

---

### TEST 07 — Vista integral

```sql
USE `ecomerce_230426`;

-- Primero crea la vista (ya está incluida en ecommerce_completo.sql)
-- Si ya la ejecutaste, solo consulta:
SELECT * FROM v_proceso_compra_completo LIMIT 10;
```

📸 **Toma captura mostrando todas las columnas**

---

### TEST 08 — Orden de productos en el tiempo

```sql
-- Primero encuentra un carrito con varios productos
SELECT carrito_id, COUNT(*) AS productos
FROM carrito_detalle
GROUP BY carrito_id
ORDER BY productos DESC LIMIT 5;

-- Usa el carrito_id que más productos tenga (reemplaza el 1)
SET @p_carrito_id = 1;

SELECT
    ROW_NUMBER() OVER (ORDER BY cd.fecha_registro) AS orden,
    p.nombre AS producto,
    cd.fecha_registro AS hora_seleccion,
    fn_duracion_sesion(
        LAG(cd.fecha_registro) OVER (ORDER BY cd.fecha_registro),
        cd.fecha_registro
    ) AS tiempo_desde_anterior
FROM carrito_detalle cd
JOIN productos p ON p.id = cd.producto_id
WHERE cd.carrito_id = @p_carrito_id
ORDER BY cd.fecha_registro;
```

📸 **Toma captura — verifica que `hora_seleccion` sea creciente**

---

### TEST 09 — Métricas de tiempo

```sql
USE `ecomerce_230426`;

SELECT
    s.id AS sesion,
    fn_duracion_sesion(s.fecha_inicio, s.fecha_fin) AS duracion_sesion,
    fn_duracion_sesion(MIN(cd.fecha_registro), MAX(cd.fecha_registro)) AS tiempo_eligiendo,
    COUNT(cd.producto_id) AS productos_elegidos
FROM sesiones s
JOIN carritos c ON c.sesion_id = s.id
JOIN carrito_detalle cd ON cd.carrito_id = c.id
WHERE s.fecha_fin IS NOT NULL
GROUP BY s.id, s.fecha_inicio, s.fecha_fin
ORDER BY s.id DESC LIMIT 20;
```

📸 **Toma captura**

---

### TEST 10 — Dashboard de pagos (Power BI)

**Paso 1 — Obtener los datos en Workbench:**
```sql
USE `ecomerce_230426`;

SELECT
    origen AS origen_pago,
    COUNT(*) AS num_pagos,
    SUM(monto) AS total_cobrado,
    ROUND(AVG(monto), 2) AS promedio_por_pago
FROM transacciones_financieras
WHERE estatus = 'Aprobado'
GROUP BY origen
ORDER BY total_cobrado DESC;
```

**Paso 2 — Exportar a CSV:**
1. Ejecuta la consulta anterior
2. En el panel de resultados, haz clic en el ícono de exportar (parece un diskette con flecha)
3. Guarda el archivo como `pagos_por_origen.csv` en la carpeta `/Tests`

**Paso 3 — Importar en Power BI:**
1. Abre **Power BI Desktop**
2. Clic en **"Obtener datos"** → **"Texto/CSV"**
3. Selecciona el archivo `pagos_por_origen.csv`
4. Clic en **"Cargar"**
5. En el panel derecho, arrastra `origen_pago` al eje X
6. Arrastra `total_cobrado` a los valores
7. Elige el tipo de gráfica que prefieras (barras o pastel)

📸 **Toma captura del dashboard**

---

## Cómo verificar que los datos son correctos

Después de ejecutar las simulaciones, corre estas consultas de verificación. **Deben devolver 0 filas**. Si devuelven filas, hay un problema.

```sql
USE `ecomerce_230426`;

-- ¿Alguna sesión tiene hora de fin antes que la de inicio? (debe ser 0)
SELECT COUNT(*) AS sesiones_con_error
FROM sesiones
WHERE fecha_fin IS NOT NULL AND fecha_fin <= fecha_inicio;

-- ¿Alguna sesión tiene hora de inicio igual al registro? (deben ser distintas)
SELECT COUNT(*) AS sesiones_sin_desfase
FROM sesiones
WHERE fecha_inicio = fecha_registro;

-- ¿El carrito muestra la hora del último producto? (debe ser 0)
SELECT COUNT(*) AS carritos_desactualizados
FROM carritos c
JOIN carrito_detalle cd ON cd.carrito_id = c.id
GROUP BY c.id, c.fecha_actualizacion
HAVING c.fecha_actualizacion < MAX(cd.fecha_registro);
```

Si todas devuelven `0` — los datos están perfectos ✅

---

## Solución de problemas frecuentes

### ❌ Error: "Table 'ecomerce_230426.X' doesn't exist"
**Causa:** La tabla X no existe en la BD del docente o tiene otro nombre.
**Solución:** Ejecuta `SHOW TABLES;` para ver todas las tablas disponibles y compara con el nombre que usa el script.

### ❌ Error: "Unknown column 'X' in field list"
**Causa:** Una columna tiene un nombre diferente en el esquema real.
**Solución:** Ejecuta `DESCRIBE nombre_tabla;` para ver las columnas exactas.

### ❌ El procedimiento tarda demasiado (Test 06)
**Causa:** 10,000 inserciones toman tiempo.
**Solución:** Antes de ejecutar, corre:
```sql
SET SESSION wait_timeout = 600;
SET SESSION interactive_timeout = 600;
```

### ❌ Error: "Duplicate entry for key PRIMARY"
**Causa:** Se está intentando insertar el mismo producto dos veces en el mismo carrito.
**Solución:** El SP ya maneja esto con una tabla temporal aleatoria. Si el error persiste, verifica que haya suficientes productos activos en el catálogo para la categoría seleccionada:
```sql
SELECT COUNT(*) FROM productos WHERE estatus = 'Activo';
```

### ❌ Error al crear índice: "Duplicate key name"
**Causa:** El índice ya existe de una ejecución anterior.
**Solución:** El script ya incluye `DROP INDEX IF EXISTS` antes de cada `CREATE INDEX`. Si aún falla, ejecuta manualmente:
```sql
DROP INDEX nombre_del_indice ON nombre_tabla;
```

---

## Glosario rápido

| Término | ¿Qué significa? |
|---|---|
| **Esquema** | Una base de datos. `ecomerce_230426` es nuestro esquema |
| **Tabla** | Como una hoja de Excel con filas y columnas |
| **PK (Primary Key)** | El ID único de cada fila — no puede repetirse |
| **FK (Foreign Key)** | Un campo que apunta al ID de otra tabla |
| **Trigger** | Código que se ejecuta solo cuando ocurre algo en una tabla |
| **Procedimiento (SP)** | Una receta de pasos que ejecutamos con `CALL` |
| **Función** | Como un procedimiento pero devuelve un valor calculado |
| **Índice** | Acelera las búsquedas, como el índice de un libro |
| **Transacción** | Un grupo de pasos que se hacen todos juntos o no se hace ninguno |
| **COMMIT** | Confirmar y guardar los cambios de una transacción |
| **ROLLBACK** | Cancelar y deshacer los cambios si algo salió mal |
| **Bitácora** | Historial automático de todo lo que pasó en la BD |
| **Timestamp** | Una fecha con hora exacta (ej. `2026-06-15 11:43:00`) |

---

## Resumen de lo que se entrega

| Archivo | Dónde está | Qué contiene |
|---|---|---|
| `ecommerce_completo.sql` | `/BD` | Todo el código SQL en un solo archivo |
| `diccionario_datos.md` | `/BD` | Explicación de cada tabla y columna |
| `log_prompts_IA.md` | `/Asistencia_IA` | Registro de uso de IA (convertir a PDF) |
| `GUIA_INTEGRACION_Y_PRUEBAS.md` | raíz | Guía técnica detallada |
| `DOCUMENTACION_GENERAL.md` | raíz | Esta documentación general |
| Capturas de pantalla | `/Tests` | Las toman los integrantes al ejecutar |
| DER y diagrama relacional | `/BD` | Se generan desde Workbench |
| `README.md` | raíz | Portada del repositorio GitHub |

---

*Proyecto desarrollado con apoyo de Claude Code (Anthropic) — Marzo 2026*
