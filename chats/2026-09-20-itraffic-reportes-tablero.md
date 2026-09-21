# itraffic_connector: reportes y tablero alineados a iTraffic

Fecha: 2026-09-20. Continúa
[`2026-09-12-itraffic-connector-modulo.md`](./2026-09-12-itraffic-connector-modulo.md).

## Objetivo del pedido

Que las visualizaciones y reportes de Odoo sean **exactos a los de
iTraffic** (mismos filtros, mismas columnas, mismos cortes), con un tablero
de lectura rápida, para auditoría, caja, proveedores y clientes.

## Hallazgo 1 (crítico): fechas vacías se mandaban como 1900-01-01

Un campo `Date` vacío en Odoo vale `False`, no `None`. `pymssql` serializa
`False` como `0`, que SQL Server interpreta como `1900-01-01` — o sea, un
filtro de fecha activo que no matchea nada. Síntoma: reportes que devuelven
**cero filas sin error**, o que devuelven muchas menos de las que
corresponden.

Se vio en vivo: Caja pasó de 8 a 213 filas al corregirlo; Proveedores y
Autorizaciones pasaron de 0 filas a 199 y 35.

**Regla**: todo campo de fecha/valor opcional de Odoo tiene que viajar al
SP como `campo or None`, nunca directo. Está centralizado en
`ItrafficQuery._common_params()`.

## Hallazgo 2: `@tipocc` es el "modo de corte" del reporte

Los SP de saldo aceptan `@tipocc`, que es el conmutador que usa cada
variante del reporte en iTraffic (el mismo SP sirve a decenas de títulos
distintos de `Informesweb`). Verificado empíricamente contra la base real
(no deducido del código, que tiene miles de líneas de ramas):

| tipocc | Qué devuelve | Con qué filtro de fecha funciona |
|---|---|---|
| `1` | Detalle por reserva: trae `rva`, vendedor, pax, estado de reserva, `nro_conf` | **Fecha de viaje** (`@fec_Saldesde/Salhasta`) |
| `2` | Resumido por comprobante: una línea por comprobante, sin datos de reserva en Proveedores | **Fecha de comprobante** (`@fec_Compdesde/CompHasta`) |
| `15` | Facturas con saldo pendiente — equivale a "Facturas a pagar" | Cualquiera de las dos |

Lo importante: **los modos 1 y 2 son mutuamente excluyentes en el filtro de
fecha**. Mandarle a tipocc=1 un rango de fecha de comprobante devuelve cero
filas, y viceversa. Por eso en Odoo, cuando se elige "Detalle por reserva",
el rango principal se manda al slot de fecha de viaje (y la pantalla lo
avisa en amarillo).

En `iLSALDAUTORIZA_ListItraffic` el `tipocc` **no cambia nada** y solo
funciona el filtro por fecha de comprobante — por eso ahí queda fijo en 2 y
no se le ofrece el selector al usuario.

## Hallazgo 3: `Informesweb.Nombre` es el `@namereport`

La columna `Nombre` del catálogo (`SALDOPROVEEDOR1`,
`SALDOPROVEEDOR2_FACTURAS`, `SALDOAUTORIZADIARIO`, ...) es el valor que el
ERP pasa como `@namereport` al SP, y el SP ramifica sobre él. La columna
`Filtros` está vacía para estos reportes: los filtros reales son los
parámetros del SP, no un metadato del catálogo.

## Hallazgo 4: la auditoría real es `dbo.AuditLog`

- `dbo.AuditLog`: **viva y con ~2 millones de filas**, escribiendo en el
  momento. La alimenta el SP `Common_AuditLog`. Columnas: `UserId`,
  `UserName`, `Action`, `ChangedOn`, `TableName`, `RowId`, `Module`,
  `Page`, `Changes`, `RowIdParent`, `TableNameParent`.
- `Changes` es JSON: `[{"F": campo, "O": valor viejo, "V": valor nuevo}]`.
- Acciones observadas: `INSERT`, `UPDATE`, `DELETE`, `PRINT`.
- `dbo.Logsistema` y `dbo.ReservaAuditoria` **existen pero están vacías** —
  son legado, no usarlas.

## Qué quedó construido

### Tipo de reporte nuevo: Auditoría

Modelo `itraffic.auditoria.line` sobre `dbo.AuditLog`. Filtros: fechas,
usuario, acción, tabla (coincidencia parcial) e ID de registro (matchea
tanto `RowId` como `RowIdParent`, para seguir un registro y sus hijos).
El JSON de `Changes` se parsea a `campos_modificados` y `cantidad_campos`;
si no parsea, se deja vacío en vez de romper la consulta entera.

### Antigüedad de saldo (aging)

`itraffic.aging.mixin`: `dias_para_vencer`, `estado_vencimiento`
(vencido / vence en 7 días / sin urgencia / sin vencimiento) y
`tramo_antiguedad` (a vencer, 1-30, 31-60, 61-90, +90). El modelo concreto
declara en `_vencimiento_field` cuál de sus fechas es el vencimiento,
porque cada SP lo llama distinto (`fec_vencop` en proveedores y
autorizaciones, `Fec_Vence` en clientes). Lo usan Proveedores, Clientes y
Autorizaciones, y es la columna del pivot por defecto.

### Filtros ahora expuestos (los que ya tenía el SP y no se usaban)

Fechas de viaje, de reserva, de vencimiento y de check-in/check-out como
filtros **independientes** entre sí; código de vendedor; Nº de reserva;
estado de reserva; forma de pago; cuenta contable; cuenta de caja; usuario;
sucursal; y un tope de filas configurable por consulta.

`@whereExpr` y `@orderExpr` siguen **sin alimentarse nunca** (SQL dinámico
del SP). El comodín `LIKE` del filtro de tabla en auditoría se arma en
Python y viaja como parámetro ligado, no concatenado.

### Columnas

Cada grilla trae ahora el juego completo de columnas que devuelve el SP
(30 a 44 campos según el reporte), con `optional="hide"` en las
secundarias: el usuario las prende desde el selector de columnas, igual que
elige columnas en iTraffic, sin que la grilla por defecto sea ilegible.

Nota: con `tipocc=2` varias columnas de nivel reserva (`rva`, vendedor,
pax, estado) vienen **vacías desde el SP** — no es un error de mapeo. Para
verlas hay que usar "Detalle por reserva".

### Tablero

- Banda de totales en el formulario (filas, saldo total, saldo vencido,
  débitos, créditos), visible sin abrir el gráfico.
- Totales por columna (`sum=`) en las grillas.
- Vistas de búsqueda por modelo con filtros de un clic (Vencido, Vence en 7
  días, Con saldo, Saldado, Pesos/Dólares, Ingresos/Egresos, Altas/Bajas/
  Modificaciones) y agrupaciones listas (proveedor, cliente, moneda, tramo
  de antigüedad, forma de pago, sucursal, vendedor, centro de costo,
  usuario, mes de vencimiento/comprobante/viaje).
- Gráficos por defecto orientados a decisión: saldo apilado por tramo de
  antigüedad (proveedores y clientes), débito/crédito por día y forma de
  pago (caja), proyección de vencimientos por semana (autorizaciones),
  actividad por día y acción (auditoría).

## Cómo se validó

`odoo shell` contra la base real, rango corto (15 al 20 de septiembre de
2026), `limit_rows=200/300`. Los cinco reportes devuelven datos:
proveedores 199, clientes 299, caja 213, autorizaciones 35, auditoría 300.
Las 8 combinaciones de modelo × vista (form/list/search/graph/pivot) se
validaron con `get_views`.

## Anexo: el acceso directo del escritorio no levantaba Odoo

El `.lnk` del escritorio apunta bien a
`D:\Repos\Odoo HR\2-Abrir Odoo RRHH.bat`, y el `.bat` tenía CRLF correcto.
Eran tres bugs dentro del script, que solo se disparaban **con Docker
Desktop apagado** (por eso no se habían visto):

1. **Ruta de Docker Desktop hardcodeada** en `C:\Program Files\Docker\...`.
   En esta PC está instalado a nivel usuario, en
   `%LOCALAPPDATA%\Programs\DockerDesktop\`. Ahora se buscan las dos.
2. **Etiquetas `:label` dentro de un bloque `if ( ... )`** de cmd, más
   `%ProgramFiles%` expandido dentro de ese bloque. Si la ruta trae
   paréntesis, el `)` corta el bloque antes de tiempo y cmd tira "No se
   esperaba ... en este momento". Se aplanó el flujo con `goto`, sin
   bloques que envuelvan etiquetas.
3. **Esperaba HTTP 200 exacto** de `/web/login`, que en realidad contesta
   **303** (redirección). Odoo levantaba bien y el script igual reportaba
   "no respondió después de 3 minutos". Ahora usa `curl -L` y acepta
   cualquier respuesta que no sea `000`.

También se cambió `timeout /t` por `ping -n` en las esperas: `timeout`
falla si el `.bat` corre con la entrada redirigida (otro script, Programador
de tareas) y llena la pantalla de errores.

Validado apagando Docker Desktop y corriendo el `.bat` de punta a punta:
arranca Docker, levanta los contenedores, detecta Odoo y abre el navegador,
con código de salida 0.

## Anexo 2: "en autorizaciones aparece todo vencido aunque haya pagos"

Reporte del usuario, y era real. Dos causas distintas.

### Causa 1: faltaba cruzar la autorización con su orden de pago

`iLSALDAUTORIZA_ListItraffic` devuelve **un movimiento por fila**, no el
saldo neto:

- `TipoComp = 'AUT'` → la autorización (`autorizado` > 0, `pagos` = 0)
- `TipoComp = 'O/P'` → la orden de pago que la cancela (`autorizado` = 0,
  `pagos` > 0, `saldo` negativo)

**La clave del cruce es `NroComp`**, que es el número de autorización y lo
comparten las dos filas. Ojo con no confundirlo: `Nro_comp` (con guion
bajo) es el número propio de cada documento y es **distinto** en cada fila
— cruzar por ese campo no une nada. El código viejo mapeaba
`Nro_comp` primero y perdía el vínculo.

Medido sobre agosto-septiembre 2026: 587 filas → 297 grupos, de los cuales
**290 son pares AUT+O/P exactos** y 7 quedan con una sola fila. Solo 4
órdenes de pago tocan más de una autorización, y aun así el SP las devuelve
como filas separadas por autorización, así que netear por `NroComp` no
mezcla importes de autorizaciones distintas.

Ahora el reporte muestra **una línea por autorización** con `autorizado`,
`pagos` y `saldo` neteados, más `nro_comp_pago`, `fecha_pago` y un
`estado_pago` (pendiente / parcial / pagado / pagado de más / pago sin
autorización). Queda un check "Ver movimientos sin cruzar" para volver a la
vista cruda del ERP.

### Causa 2: "vencido" no miraba el saldo

`_compute_aging` marcaba vencido cualquier fila con fecha de vencimiento
pasada, **sin importar si quedaba algo por pagar**. Corregido: si el saldo
es cero → `saldado`; si es negativo → `a_favor`; solo se evalúa
vencimiento cuando realmente hay saldo pendiente. Esto también arregla
Proveedores y Clientes, donde las filas de pago (O/P, REC) aparecían todas
en rojo.

Resultado sobre la misma consulta de prueba: de 35 filas "todas vencidas" a
18 autorizaciones, con **1 sola vencida de verdad**, 10 saldadas y 7
marcadas para revisar.

### Anomalía real de datos, no la tapamos

Hay autorizaciones donde el pago supera lo autorizado, a veces por mucho
(ej.: autorización 2841 = 1.557.956,16 pagada con 12.463.649,28; la 2850
pagada exactamente al doble). Se verificó que **no** es un artefacto del
cruce: el grupo tiene exactamente 2 filas, misma moneda y mismo tipo de
cambio, y ninguna otra autorización comparte esa orden de pago. Es dato del
ERP. Por eso esos casos se marcan como "Pagado de más (revisar)" y salen en
rojo, en vez de disimularlos con un saldo neto raro. **Falta que alguien
que use la pantalla de iTraffic confirme qué significan.**

### Detalle técnico

`_num()` devolvía el `Decimal` crudo del driver cuando el valor no era
cero, y `0.0` (float) cuando sí. Al sumarlos: `unsupported operand type(s)
for +: 'decimal.Decimal' and 'float'`. Ahora siempre castea a `float`.

## Pendiente

- Confirmar con un usuario de iTraffic los casos "pagado de más".
- Lo ya anotado en la memoria consolidada sigue igual (Ganancias 4ta,
  `iLSALDRVA_ListItraffic_Prevision`, Tarifario Hotel, Rentabilidad por
  File).
- `@tiposaldo` se muestra como columna pero **no se ofrece como filtro**:
  no se comprobó qué valores acepta como entrada y no se adivinó.
- Los tipocc `3, 4, 5, 6, 7, 11, 14, 20, 90, 99` existen en el SP y no se
  probaron — si hace falta otra variante de iTraffic, medir primero contra
  la base antes de ofrecerla.
- La verificación fue por `odoo shell` y `get_views`, no haciendo clic en
  la interfaz. Vale una pasada manual por la pantalla.
