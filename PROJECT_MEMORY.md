# Memoria consolidada del proyecto Odoo HR

Última consolidación: 2026-09-12
Fuentes: 4 documentos en [`chats/`](./chats/)
Naturaleza: guía operativa y mapa de conocimiento; no reemplaza el código
ni una consulta actual a Odoo/SQL Server.

## 1. Cómo leer esta memoria

Distingue: **Confirmado en código/datos** (observado directamente),
**Decisión funcional** (elegida explícitamente por el usuario, vigente
hasta que se cambie), y **Pendiente/hipótesis** (no habilita cambios
productivos sin validar). Los conteos, IDs y valores de ejemplo son
fotografías históricas — reconfirmar contra el sistema real antes de
asumir que siguen vigentes.

## 2. Contexto y arquitectura

`Odoo HR` (`D:\Repos\Odoo HR`) es un proyecto Odoo 18 Community corriendo
en Docker Compose (Postgres + Odoo, imagen propia `Dockerfile.odoo` que
extiende `odoo:18.0` agregando el driver `pymssql`). Base de datos
principal en uso: **`RRHH_CuencaDelPlata`**. Hay otra base, `odoohr`, que
quedó desincronizada de un intento viejo — no es donde vive el trabajo
real.

Tres módulos propios conviven en `custom-addons/`:

| Módulo | Qué hace | Memoria detallada |
|---|---|---|
| `hr_argentina_core` | RR.HH. argentino: CCT, empleado, nómina propia, quejas/denuncias, chatbot+IA | [`2026-09-12-hr-argentina-core-modulo.md`](./chats/2026-09-12-hr-argentina-core-modulo.md) |
| `maxirest_connector` | Conector de demo hacia MaxiRest (sin API real disponible) | [`2026-09-12-maxirest-connector-modulo.md`](./chats/2026-09-12-maxirest-connector-modulo.md) |
| `itraffic_connector` | Consultas de solo lectura contra la base SQL Server real de iTraffic (reservas, proveedores, clientes, caja, autorizaciones, stock) | [`2026-09-12-itraffic-connector-modulo.md`](./chats/2026-09-12-itraffic-connector-modulo.md) |

Además hay datos de demo de la empresa "Cuenca del Plata" cargados en
`hr_argentina_core` — ver
[`2026-09-12-organigrama-cuenca-del-plata.md`](./chats/2026-09-12-organigrama-cuenca-del-plata.md).

## 3. Principios operativos consolidados

### Seguridad y credenciales

- **Nunca se escriben credenciales reales en archivos versionados de
  código** (ni de `Odoo HR`, ni de `Odoo HR Context`). Las credenciales de
  integraciones externas (IA de Anthropic, SQL Server de iTraffic) viven
  como `ir.config_parameter` dentro de Odoo, cargadas desde la pantalla de
  Ajustes de cada módulo — nunca hardcodeadas.
- Todas las consultas a bases externas son de **solo lectura**. Los
  conectores construidos (`itraffic_connector`) usan parámetros ligados
  (nunca concatenación de texto) específicamente para evitar inyección
  SQL, y nunca alimentan los parámetros `@whereExpr`/`@orderExpr` que
  algunos SP de iTraffic exponen para SQL dinámico.
- Antes de dar por buena una integración externa, verificar primero si
  existe una API pública documentada (no asumir) — el caso de MaxiRest
  demostró que no siempre la hay, y ahí corresponde un conector de
  demostración en vez de inventar una integración real inexistente.

### Trabajo con Odoo 18

- Las vistas de lista usan `<list>`, no `<tree>` (cambió en v17/18).
- El validador de vistas de Odoo 18 no acepta `active_id` como variable de
  contexto dentro de un botón en el arch — usar `id`.
- Las decoraciones de vistas (`decoration-danger`, etc.) no soportan
  funciones Python arbitrarias como `abs()` — si se rompe el render de una
  grilla sin error de servidor visible, sospechar primero de la
  decoración.
- `pymssql` no viene en la imagen oficial `odoo:18.0` — hace falta un
  Dockerfile propio para instalarlo de forma persistente.

### Trabajo con datos externos reales (iTraffic / SQL Server)

- Antes de reconstruir la lógica de un reporte desde cero, buscar si ya
  existe como stored procedure real usado por el ERP — la tabla
  `dbo.Informesweb` en la base de iTraffic es el catálogo oficial de
  reportes con su SP asociado.
- Ante una consulta que tarda demasiado o parece colgada: revisar
  `sys.dm_exec_requests` en SQL Server (para ver si la base real está
  trabajando) y `pg_stat_activity` en Postgres (para encontrar una
  transacción de Odoo colgada bloqueando el sistema) antes de asumir que
  el problema es la base externa.

## 4. Pendientes priorizados

1. Impuesto a las Ganancias 4ta categoría en `hr_argentina_core` (requiere
   modelo de escalas con vigencia por fecha).
2. Reporte "Saldo con cliente proyectado" en `itraffic_connector`
   (`iLSALDRVA_ListItraffic_Prevision`) — no se logró hacer devolver
   datos con los parámetros probados hasta ahora.
3. Tarifario Hotel (`iTarifaHotel_ListSmart`) y Rentabilidad por File
   (`iLRESERVA_ListItrafficIngresosEgresos`) en `itraffic_connector` —
   identificados, no implementados.
4. Si se consigue acceso real a la API de MaxiRest, implementar
   `maxirest.connector.api` sin tocar el resto del módulo.
5. Si se decide activar Inventario/Compras/Facturación nativos de Odoo,
   migrar los modelos propios de `maxirest_connector` a los nativos.

## 5. Índice de fuentes

| Tema | Fuente |
|---|---|
| hr_argentina_core (CCT, nómina, quejas, chatbot/IA, seguridad) | [`2026-09-12-hr-argentina-core-modulo.md`](./chats/2026-09-12-hr-argentina-core-modulo.md) |
| maxirest_connector (sin API real, conector demo) | [`2026-09-12-maxirest-connector-modulo.md`](./chats/2026-09-12-maxirest-connector-modulo.md) |
| itraffic_connector (SQL real, catálogo Informesweb, incidente de conexión) | [`2026-09-12-itraffic-connector-modulo.md`](./chats/2026-09-12-itraffic-connector-modulo.md) |
| Organigrama de demo "Cuenca del Plata" | [`2026-09-12-organigrama-cuenca-del-plata.md`](./chats/2026-09-12-organigrama-cuenca-del-plata.md) |
