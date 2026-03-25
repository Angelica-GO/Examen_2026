# Diccionario de Datos
## Boutique E-commerce — `ecomerce_230426`
**Proyecto:** Examen Práctico BD para Negocios Digitales
**Fecha:** 2026-03-22
**Motor:** MySQL 8.0 | Charset: utf8mb4

---

## Índice de tablas

| # | Tabla | Descripción corta |
|---|---|---|
| 1 | `sesiones` | Registro de cada visita de un usuario a la plataforma |
| 2 | `carritos` | Intención de compra vinculada a una sesión |
| 3 | `carrito_detalle` | Líneas de productos dentro de un carrito |
| 4 | `pedidos` | Venta formal generada desde un carrito |
| 5 | `transacciones_financieras` | Registro del cobro de cada pedido |
| 6 | `metodos_pago` | Tarjetas y cuentas registradas por usuario |
| 7 | `usuarios` | Credenciales de acceso a la plataforma |
| 8 | `personas` | Entidad base para persona física y moral |
| 9 | `persona_fisica` | Datos personales de clientes individuales |
| 10 | `persona_moral` | Datos de empresas o personas jurídicas |
| 11 | `clientes` | Vínculo entre persona y rol de cliente |
| 12 | `productos` | Catálogo de artículos de la boutique |
| 13 | `categorias` | Clasificación de productos |
| 14 | `productos_categorias` | Relación N:M productos ↔ categorías |
| 15 | `divisas` | Monedas soportadas en transacciones |
| 16 | `proveedores` | Proveedores de productos |
| 17 | `bitacora` | Auditoría de eventos generados por triggers |

---

## Reglas de negocio globales

| Regla | Descripción |
|---|---|
| **Horario boutique** | Todas las marcas de tiempo simuladas caen entre las **10:00 y las 21:00** del día dado. El inicio de sesión se limita a 10:00–18:00 para dejar margen a los productos y al cierre. |
| **Enfoque C (fin_sesion)** | `sesiones.fecha_fin` se actualiza al **timestamp real del pago exitoso** dentro de `sp_finalizar_compra`. `sp_simular_carrito` asigna un valor provisional hasta ese momento. |
| **Timestamps crecientes** | Cada línea de `carrito_detalle` tiene `fecha_registro` estrictamente mayor que la anterior dentro del mismo carrito: t₁ < t₂ < … < tₙ. |
| **Actualización de carrito** | `carritos.fecha_actualizacion` refleja siempre el timestamp del **último producto agregado**, garantizado por el trigger `trg_carrito_detalle_ai`. |
| **PK compuesta en detalle** | `carrito_detalle` tiene PK `(carrito_id, producto_id)`: un mismo producto no puede repetirse en el mismo carrito. |

---

## 1. `sesiones`

Registra cada visita/sesión de un usuario en la plataforma. Es la tabla raíz de la línea temporal de compra.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK — Identificador único de sesión |
| `usuario_id` | INT UNSIGNED | NO | — | FK → `usuarios.persona_fisica_id` |
| `origen` | ENUM | NO | `'Plataforma'` | Canal de entrada: `Plataforma` / `Liga Externa` / `Campaña Publicitaria` |
| `tipo_dispositivo` | ENUM | NO | — | `PC` / `Laptop` / `Smart Phone` / `Tablet` / `Smart TV` / `Voice Assistant` |
| `sistema_operativo` | VARCHAR(50) | NO | — | Ej. `Windows 11`, `iOS 17`, `Android 14` |
| `codigo_pais` | CHAR(2) | NO | — | Código ISO 3166-1 alfa-2. Ej. `MX`, `US`, `ES` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Momento en que el sistema crea la sesión (login) |
| `fecha_inicio` | DATETIME | SÍ | NULL | Momento en que el usuario empieza a navegar activamente. Siempre **>= fecha_registro** (CHECK constraint). Desfase simulado: 1–3 min |
| `fecha_fin` | DATETIME | SÍ | NULL | Momento de cierre de sesión. **Enfoque C**: se asigna en `sp_finalizar_compra` al confirmarse el pago. Provisional al final de `sp_simular_carrito`. Siempre **> fecha_inicio** |
| `estatus` | ENUM | NO | `'Activa'` | `Activa` / `Finalizada` / `Cancelada` / `Suspendida` |

**Relaciones:**
- `usuario_id` → `usuarios.persona_fisica_id`
- 1 sesión puede tener 1 carrito (`carritos.sesion_id`)

**Constraint notable:**
```sql
CHECK (fecha_inicio >= fecha_registro)
```

**Función asociada:** `fn_duracion_sesion(fecha_inicio, fecha_fin)` → `HH:MM:SS`

---

## 2. `carritos`

Representa la intención de compra de un usuario durante una sesión. Se actualiza automáticamente con cada producto agregado.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK — Identificador único del carrito |
| `sesion_id` | INT UNSIGNED | SÍ | NULL | FK → `sesiones.id`. Vincula el carrito a la sesión activa |
| `divisa_id` | INT UNSIGNED | NO | `1` | FK → `divisas.id`. Moneda del carrito (por defecto MXN) |
| `total_productos` | INT UNSIGNED | NO | `0` | Cantidad total de ítems. Actualizado por `trg_carrito_detalle_ai` |
| `monto_aproximado` | DOUBLE UNSIGNED | NO | `0` | Suma de `cantidad × precio_unitario` de todas las líneas. Actualizado por trigger |
| `fecha_creacion` | DATETIME | NO | `CURRENT_TIMESTAMP` | Momento en que el carrito fue creado. Alineado con `fecha_inicio` sesión (mismo minuto o hasta 2 min después) |
| `fecha_actualizacion` | DATETIME | SÍ | NULL | Timestamp del **último producto agregado**. Actualizado por `trg_carrito_detalle_ai` en cada INSERT en `carrito_detalle`. Ejemplo: prod. a 11:43 → `11:43`; siguiente a 11:46 → `11:46` |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Abandonado` / `Cancelado` / `Convertido` / `Expirado` |

**Relaciones:**
- `sesion_id` → `sesiones.id`
- `divisa_id` → `divisas.id`
- 1 carrito tiene N líneas en `carrito_detalle`
- 1 carrito puede generar 1 pedido (`pedidos.id_carrito`)

**Trigger asociado:** `trg_carrito_detalle_ai` (AFTER INSERT en `carrito_detalle`)

---

## 3. `carrito_detalle`

Líneas de productos seleccionados dentro de un carrito. Cada fila registra el momento exacto de selección.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `carrito_id` | INT UNSIGNED | NO | — | PK + FK → `carritos.id` |
| `producto_id` | INT UNSIGNED | NO | — | PK + FK → `productos.id`. **Un producto no puede repetirse en el mismo carrito** (PK compuesta) |
| `cantidad` | INT UNSIGNED | NO | — | Unidades seleccionadas del producto |
| `precio_unitario` | DOUBLE UNSIGNED | NO | — | Precio al momento de la selección (puede diferir del precio actual del catálogo) |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | **Momento de selección del producto**. Debe ser estrictamente creciente dentro del mismo carrito. Generado por `fn_tiempo_realista` en `sp_simular_carrito` |
| `fecha_actualizacion` | DATETIME | SÍ | NULL | Última modificación de la línea (cambio de cantidad, etc.) |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Eliminado` / `Cancelado` / `Pendiente` |

**Clave primaria:** `(carrito_id, producto_id)` — compuesta

**Regla de timestamps:** t₁ < t₂ < … < tₙ para todas las líneas del mismo `carrito_id`, garantizado por `fn_tiempo_realista` (suma 2–8 min en cada llamada).

**Trigger asociado:** `trg_carrito_detalle_ai` — al insertar una línea, actualiza `carritos.fecha_actualizacion`, `total_productos` y `monto_aproximado`.

---

## 4. `pedidos`

Registro formal de la venta. Se crea desde `sp_finalizar_compra` a partir de un carrito activo.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK — Identificador único del pedido |
| `id_carrito` | INT UNSIGNED | NO | — | FK → `carritos.id`. Carrito que originó el pedido |
| `total_productos` | INT UNSIGNED | NO | `0` | Cantidad de productos en el pedido |
| `importe_total` | DECIMAL(10,2) | NO | `0.00` | Monto total del pedido en la divisa del carrito |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha y hora de creación del pedido |
| `fecha_actualizacion` | DATETIME | SÍ | NULL | Última modificación. Actualizada automáticamente por `trg_pedidos_bu` |
| `estatus` | ENUM | NO | — | `Pendiente` → `Confirmado` → `Pagado` → `En preparacion` → `Enviado` → `Entregado`. También: `Cancelado` / `Devuelto` |
| `aprobacion` | BIT(1) | NO | — | `0` = no aprobado / `1` = aprobado tras pago exitoso |

**Relaciones:**
- `id_carrito` → `carritos.id`
- 1 pedido genera 1 transacción en `transacciones_financieras`

**Triggers asociados:**
- `trg_pedidos_ai`: registra creación en bitácora
- `trg_pedidos_bu`: valida transición de estatus; actualiza `fecha_actualizacion`
- `trg_pedidos_au`: registra cambio de estatus en bitácora

---

## 5. `transacciones_financieras`

Registro del cobro de cada pedido. Contiene el origen de pago y el estatus de la aprobación.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT(10) UNSIGNED ZEROFILL | NO | AUTO_INCREMENT | PK — Identificador único de la transacción |
| `tipo` | ENUM | NO | — | `Compra` / `Venta` / `Salario` / `Renbolso` |
| `origen` | ENUM | NO | — | `Tarjeta de Debito` / `Tarjeta de Credito` / `Transferencia bancria` / `Socio Comercial` / `Tarjeta de Regalo` / `Creditos Devolucion` / `Vales Despensa` |
| `clave_interbancaria` | CHAR(18) | SÍ | NULL | CLABE interbancaria (18 dígitos) para transferencias |
| `numero_convenio` | VARCHAR(50) | SÍ | NULL | Número de convenio para pagos a través de socio comercial |
| `monto` | DECIMAL(10,2) | NO | — | Importe de la transacción. Debe ser > 0 (validado por `trg_trans_bi`) |
| `metodo_pago_id` | INT(10) UNSIGNED ZEROFILL | SÍ | NULL | FK → `metodos_pago.id`. Método utilizado para el pago |
| `estatus` | ENUM | SÍ | NULL | `Aprobado` / `No Aprobado` / `Pendiente` / `Cancelado` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha y hora de la transacción |
| `fecha_actualizacion` | VARCHAR(45) | SÍ | NULL | Última modificación del registro |

**Nota de diseño:** No existe FK directa hacia `pedidos`. La actualización de `pedidos.estatus` a `'Pagado'` se realiza explícitamente en `sp_finalizar_compra` (paso 2b), no desde el trigger.

**Triggers asociados:**
- `trg_trans_bi`: valida monto > 0 y método de pago vigente
- `trg_trans_ai`: registra ingreso en bitácora

---

## 6. `metodos_pago`

Tarjetas y cuentas bancarias registradas por cada usuario.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `usuario_id` | INT UNSIGNED | NO | — | FK → `usuarios.persona_fisica_id` |
| `numero_tarjeta` | CHAR(16) | NO | — | 16 dígitos numéricos. Validado por `trg_metodos_pago_bi` |
| `tipo_red_bancaria` | ENUM | NO | — | `Visa` / `MasterCard` / `American Express` |
| `tipo_cuenta` | ENUM | NO | — | `Nomina` / `Debito` / `Ahorro` / `Digital` |
| `titular` | VARCHAR(100) | NO | — | Nombre del titular como aparece en la tarjeta |
| `fecha_expiracion` | CHAR(5) | NO | — | Formato `MM/YY`. Validado por `trg_metodos_pago_bi` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha de alta del método de pago |
| `fecha_actualizacion` | DATETIME | SÍ | NULL | Última modificación |
| `estatus` | ENUM | SÍ | `'Vigente'` | `Vigente` / `Expirada` / `Desactivada` |

**Triggers asociados:**
- `trg_metodos_pago_bi`: valida 16 dígitos y formato MM/YY antes de insertar
- `trg_metodos_pago_au`: auditoría en bitácora al actualizar
- `trg_metodos_pago_ad`: auditoría en bitácora al eliminar

---

## 7. `usuarios`

Credenciales de acceso a la plataforma. PK es el mismo ID de `persona_fisica`.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `persona_fisica_id` | INT UNSIGNED | NO | — | PK + FK → `persona_fisica.id` |
| `nickname` | VARCHAR(80) | NO | — | Nombre de usuario único en la plataforma |
| `contrasenia` | VARCHAR(300) | NO | — | Contraseña (almacenada con hash) |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha de creación de la cuenta |
| `fecha_ultimo_ingreso` | DATETIME | SÍ | NULL | Último login registrado |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` / `Bloqueado` / `Cancelado` |

---

## 8. `personas`

Entidad base polimórfica para persona física y moral.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `tipo` | ENUM | NO | — | `Física` / `Moral` — determina la tabla hija |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Alta en el sistema |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` / `Eliminado` |

---

## 9. `persona_fisica`

Datos personales de clientes individuales. Hereda de `personas`.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | — | PK + FK → `personas.id` |
| `nombre` | VARCHAR(100) | NO | — | Nombre(s) del cliente |
| `apellido_paterno` | VARCHAR(100) | NO | — | Primer apellido |
| `apellido_materno` | VARCHAR(100) | SÍ | NULL | Segundo apellido (opcional) |
| `fecha_nacimiento` | DATE | SÍ | NULL | Fecha de nacimiento |
| `genero` | ENUM | SÍ | NULL | `Masculino` / `Femenino` / `No especificado` |
| `curp` | CHAR(18) | SÍ | NULL | Clave Única de Registro de Población |
| `rfc` | VARCHAR(13) | SÍ | NULL | Registro Federal de Contribuyentes |

---

## 10. `persona_moral`

Datos de empresas o personas jurídicas.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | — | PK + FK → `personas.id` |
| `razon_social` | VARCHAR(200) | NO | — | Nombre legal de la empresa |
| `rfc` | VARCHAR(13) | NO | — | RFC empresarial |
| `regimen_fiscal` | VARCHAR(100) | SÍ | NULL | Régimen fiscal ante el SAT |
| `fecha_constitucion` | DATE | SÍ | NULL | Fecha de constitución legal |

---

## 11. `clientes`

Vincula a una persona (física o moral) con el rol de cliente en la plataforma.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `persona_id` | INT UNSIGNED | NO | — | FK → `personas.id` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Alta como cliente |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` / `Bloqueado` |

---

## 12. `productos`

Catálogo completo de artículos disponibles en la boutique.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `nombre` | VARCHAR(150) | NO | — | Nombre del producto |
| `descripcion` | TEXT | SÍ | NULL | Descripción detallada |
| `marca` | VARCHAR(100) | NO | — | Marca del producto |
| `modelo` | VARCHAR(100) | SÍ | NULL | Modelo o referencia interna |
| `precio_menudeo` | DECIMAL(10,2) | NO | `0.00` | Precio unitario al consumidor final |
| `precio_mayoreo` | DECIMAL(10,2) | SÍ | NULL | Precio para compras en volumen |
| `tipo` | ENUM | NO | `'No Perecedero'` | `Perecedero` / `No Perecedero` |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Agotado` / `En existencia` / `Descontinuado` / `Baja existencia` |
| `codigo_barras` | VARCHAR(40) | NO | — | Código de barras del producto |
| `sku` | VARCHAR(40) | NO | — | Stock Keeping Unit — identificador interno |

**Uso en simulación:** `sp_simular_carrito` filtra productos con estatus `IN ('Activo','En existencia','Baja existencia')`.

---

## 13. `categorias`

Clasificación de productos por tipo (Perfumería, Ropa para mujer, Ropa para hombre, etc.).

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `nombre` | VARCHAR(100) | NO | — | Nombre de la categoría |
| `descripcion` | TEXT | SÍ | NULL | Descripción de la categoría |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha de creación |

**Uso en tests:** Tests 02, 04, 05 filtran por `categoria_id` para Perfumería y Ropa.

---

## 14. `productos_categorias`

Tabla de relación N:M entre productos y categorías. Un producto puede pertenecer a varias categorías.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `categoria_id` | INT UNSIGNED | NO | — | PK + FK → `categorias.id` |
| `producto_id` | INT UNSIGNED | NO | — | PK + FK → `productos.id` |
| `estatus` | ENUM | SÍ | `'Activo'` | `Activo` / `Inactivo` |
| `fecha_registro` | DATETIME | SÍ | `CURRENT_TIMESTAMP` | Fecha de asociación |

**Clave primaria:** `(categoria_id, producto_id)`
**Índice:** `idx_prod_cat (categoria_id)` — optimiza filtros por categoría en Tests 02, 04, 05.

---

## 15. `divisas`

Catálogo de monedas soportadas en los carritos y pedidos.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `nombre` | VARCHAR(50) | NO | — | Nombre de la moneda (ej. `Peso Mexicano`) |
| `codigo` | CHAR(3) | NO | — | Código ISO 4217 (ej. `MXN`, `USD`) |
| `simbolo` | VARCHAR(5) | SÍ | NULL | Símbolo gráfico (ej. `$`, `€`) |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` |

**Valor por defecto en carritos:** `divisa_id = 1` (Peso Mexicano).

---

## 16. `proveedores`

Catálogo de proveedores de productos para la boutique.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `persona_id` | INT UNSIGNED | NO | — | FK → `personas.id` |
| `nombre_comercial` | VARCHAR(150) | NO | — | Nombre comercial del proveedor |
| `contacto` | VARCHAR(100) | SÍ | NULL | Nombre del contacto principal |
| `telefono` | VARCHAR(20) | SÍ | NULL | Teléfono de contacto |
| `email` | VARCHAR(100) | SÍ | NULL | Correo electrónico |
| `estatus` | ENUM | NO | `'Activo'` | `Activo` / `Inactivo` / `Bloqueado` |
| `fecha_registro` | DATETIME | NO | `CURRENT_TIMESTAMP` | Fecha de alta |

---

## 17. `bitacora`

Registro de auditoría generado automáticamente por los 9 triggers del sistema.

| Columna | Tipo | Nulo | Default | Descripción |
|---|---|---|---|---|
| `id` | INT UNSIGNED | NO | AUTO_INCREMENT | PK |
| `tabla` | VARCHAR(100) | NO | — | Nombre de la tabla que originó el evento |
| `operacion` | ENUM | NO | — | `Insert` / `Update` / `Delete` |
| `usuario` | VARCHAR(100) | NO | — | Usuario de MySQL que ejecutó la operación (`USER()`) |
| `descripcion` | TEXT | SÍ | NULL | Descripción del evento con datos relevantes |
| `fecha_hora` | DATETIME | SÍ | `CURRENT_TIMESTAMP` | Timestamp del evento |

**Triggers que escriben en bitácora:**

| Trigger | Tabla origen | Operación |
|---|---|---|
| `trg_pedidos_ai` | `pedidos` | Insert |
| `trg_pedidos_au` | `pedidos` | Update |
| `trg_metodos_pago_au` | `metodos_pago` | Update |
| `trg_metodos_pago_ad` | `metodos_pago` | Delete |
| `trg_trans_ai` | `transacciones_financieras` | Insert |

**Limpieza selectiva:** solo se eliminan registros de `bitacora` donde `tabla IN ('carrito_detalle','carritos','pedidos','transacciones_financieras','sesiones')` — no se borra la bitácora de otras áreas.

---

## Funciones del sistema

| Función | Firma | Retorna | Descripción |
|---|---|---|---|
| `fn_tiempo_realista` | `(p_fecha_base DATETIME)` | `DATETIME` | Suma 2–8 minutos aleatorios a la fecha base. Simula tiempo de selección en boutique (lectura de ficha, talla, color) |
| `fn_duracion_sesion` | `(p_inicio DATETIME, p_fin DATETIME)` | `VARCHAR(8)` | Diferencia entre dos DATETIME en formato `HH:MM:SS`. Retorna `00:00:00` si alguno es NULL o fin ≤ inicio |

---

## Procedimientos almacenados

| Procedimiento | Parámetros | Descripción |
|---|---|---|
| `sp_simular_carrito` | `p_num_compras INT, p_categoria_id INT UNSIGNED, p_fecha_base DATE` | Genera N compras simuladas con lógica temporal coherente (sesión → carrito → detalle). Usa `fn_tiempo_realista` y el trigger `trg_carrito_detalle_ai` |
| `sp_finalizar_compra` | `p_carrito_id INT UNSIGNED, p_metodo_pago_id INT UNSIGNED` | Transforma carrito activo en compra completada. Incluye START TRANSACTION / COMMIT / ROLLBACK. Actualiza `fecha_fin` de sesión (Enfoque C) |

---

## Índices implementados

| Nombre | Tabla | Columna(s) | Justificación |
|---|---|---|---|
| `idx_sesiones_usuario` | `sesiones` | `usuario_id` | JOIN frecuente para obtener nombre del comprador (Tests 07, 09) |
| `idx_sesiones_fechas` | `sesiones` | `fecha_inicio, fecha_fin` | `fn_duracion_sesion` y filtros de rango horario (Tests 07, 08, 09) |
| `idx_detalle_carrito_fecha` | `carrito_detalle` | `carrito_id, fecha_registro` | ORDER BY cronológico sin filesort en Tests 08, 09 |
| `idx_pedidos_fecha` | `pedidos` | `fecha_registro` | Filtro `YEAR(fecha_registro) = 2026` en Test 03 |
| `idx_prod_cat` | `productos_categorias` | `categoria_id` | EXISTS en `sp_simular_carrito` para filtrar por categoría (Tests 02, 04, 05) |

---

## Vista

| Vista | Descripción | Test |
|---|---|---|
| `v_proceso_compra_completo` | Proceso completo sesión → carrito → pedido → pago con métricas de duración | Test 07 |

---

*Fin del diccionario de datos — `ecomerce_230426`*
