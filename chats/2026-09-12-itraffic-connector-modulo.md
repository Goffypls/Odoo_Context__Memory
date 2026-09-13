# Módulo itraffic_connector

## Objetivo y alcance

El usuario tiene un repo `D:\Repos\WSBridge-Smart` (webservice SOAP/ASMX,
.NET, llamado "BridgeService"/"SmartService") que apunta al ERP **iTraffic**
(el mismo ERP de fondo que usa Softur.Serenity — SQL Server). El pedido
final fue traer reservas, tarifas, cobros, estado de cuenta de proveedores
y clientes, caja y autorizaciones a un módulo de Odoo con grilla + botón
para ver como gráfico/pivot (estilo "PowerBI casero" con vistas nativas de
Odoo).

## Decisión de arquitectura: SQL directo, no el webservice SOAP

Se probó primero el webservice SOAP real
(`https://cuenca.itraffic.com.ar/iTraffic_Cuenca.WSBridge/BridgeService.asmx`,
credenciales de prueba `WSCONBD/WSCONBD`). Se resolvió un bug real de
namespaces XML (los elementos hijos de un `xsstring8`/`getBookingListRQ`
deben quedar en el namespace vacío — "unqualified" — no heredar el
namespace del elemento raíz vía `xmlns` por defecto; hay que usar un
prefijo en el elemento raíz en vez de `xmlns` default). Con eso resuelto,
el servidor respondió limpio pero **rechazó esas credenciales puntuales**
("Usuario no existente o clave inválida").

El usuario ofreció y compartió credenciales de **acceso directo a la base
SQL Server real** (`itraffic_Cuenca`, un servidor con IP interna). Se
decidió pivotar a SQL directo porque:
- Es más rico que el webservice (permite usar los stored procedures reales
  del ERP, con exactamente los mismos filtros que ve un usuario).
- El estado de cuenta de proveedores **no existe como método del
  webservice** — solo como SP directo en la base.

**Regla de seguridad vigente**: la contraseña real de esa base **nunca**
se escribió en el repo de código (`Bridge.ws/Bin/TrafficDBInfo.xml` está
versionado en git y ya tenía el mal hábito de guardar passwords de otros
servidores en texto plano — no se replicó ese patrón). La credencial vive
únicamente como `ir.config_parameter` de Odoo (`itraffic_connector.server`,
`.database`, `.user`, `.password`), cargada por el usuario o por Claude
directamente vía shell con su consentimiento explícito para pruebas — no
se pegó nunca en un archivo del repositorio de código versionado.

## Hallazgo clave: tabla `Informesweb`

El ERP tiene una tabla `dbo.Informesweb` (fuente en
`Softur.Serene.Model/Migrations/DefaultDB/ScriptsBD/Tables/Informesweb.sql`
del repo `D:\Repos\Softur.Serenity`) que es el catálogo real de reportes
del sistema: cada fila tiene `Titulo`, `sp` (el stored procedure real que
arma el reporte) y `Filtros`. Consultarla en vivo (`SELECT * FROM
dbo.Informesweb WHERE Activo=1`) da el mapa exacto de qué SP usa cada
pantalla real de iTraffic — evita adivinar filtros o inventar lógica.

## Reportes construidos (todos con el mismo SP real que usa el ERP)

| Tipo en Odoo | SP real | Notas |
|---|---|---|
| `reserva` | consulta directa a `dbo.Reserva` | No usa un SP dedicado, es la tabla maestra |
| `proveedor` | `ilsaldpro_ListItraffic` | "Informe Saldo Proveedor" / "Facturas a pagar" |
| `cliente` | `iLSALDRVA_ListItraffic` | "Estado De Cuenta" / "Informe Saldo Clientes" |
| `caja` | `iLLISCAJA_ListItraffic` | "Informe Caja" / "Caja Sucursal" |
| `autorizacion` | `iLSALDAUTORIZA_ListItraffic` | "Informe Saldo Autorizaciones" / "Proyección Diaria de Pagos" — vencimientos de cupones/tarjetas (`fec_vencop`/`fec_vencad` vs. `saldo`) |
| `stock` | `sp_ControlStock_ReporteStockActual` | El más simple, un solo parámetro |

Todos los SP de saldo (`proveedor`, `cliente`, `autorizacion`) comparten la
misma familia de parámetros: `@Regcount OUTPUT`, `@skip`/`@take`
(paginación), `@whereExpr`/`@orderExpr` (SQL dinámico — **nunca se
alimentan**, siempre `NULL`, para no exponer inyección SQL), y filtros
tipados: `fec_Compdesde/hasta` (fecha de comprobante), `fec_Saldesde/hasta`
(fecha de salida = fecha de viaje — filtro DISTINTO al de comprobante),
`fec_Vencdesde/hasta`, `tipocc` (se usa `2`, el mismo valor por defecto que
usa el ERP legado según comentarios en el propio SP), `tiposaldo`,
`moneda`.

Filtro **"solo con saldo pendiente"**: no existe como parámetro de estos
SP concretos (`@lconsaldo` es de una familia hermana,
`iLSALDPROV_ForOPA_FPP_Itraffic`, usada por la pantalla de autorización de
pagos, no por estos). Se implementó como filtro Python posterior al fetch
(descarta filas con `abs(saldo) < 0.01`) — deliberado, no un descuido.

## Pendiente / no resuelto

- **"Saldo con cliente proyectado"** (`iLSALDRVA_ListItraffic_Prevision`):
  se probó con `fec_Compdesde/hasta` y con `fec_Vencdesde/hasta`, ambas
  veces sin filas ni error — no se adivinó un tercer combo de parámetros a
  ciegas. Si se retoma, hay que revisar el código fuente completo del SP
  (7000+ líneas, ver `Softur.Serene.Model/Migrations/DefaultDB/ScriptsBD/Sp/`)
  o preguntarle a alguien que use esa pantalla en el ERP qué filtros carga.
- **Tarifario Hotel** (`iTarifaHotel_ListSmart`) y **Rentabilidad por
  File** (`iLRESERVA_ListItrafficIngresosEgresos`) — identificados en el
  catálogo, todavía no implementados como tipo de reporte.
- El campo `gananciaTotal` de `dbo.Reserva` puede traer valores corruptos
  puntuales (se vio un caso con ~852 millones de "ganancia" en una sola
  reserva) — la grilla de Reservas marca en rojo esos casos, pero no hay
  que confiar en el total sin filtrar.

## Incidente: conexión colgada y lock en Postgres

Una consulta de prueba con rango de fechas muy amplio (8 meses) sobre
`ilsaldpro_ListItraffic` quedó colgada más de una hora (ni siquiera
llegó a producir el traceback inicial de arranque del script). Se
verificó con `sys.dm_exec_requests` en SQL Server que **no había ninguna
consulta corriendo del lado de iTraffic** — el problema era 100% del lado
del cliente (probablemente la conexión TCP se cortó silenciosamente sin
que pymssql lo detectara). Esa transacción quedó "idle in transaction" en
Postgres (la base de Odoo), bloqueando cualquier `ALTER TABLE` posterior
(se vio como `ERROR: canceling statement due to lock timeout` al intentar
actualizar el módulo). Se resolvió matando el backend de Postgres
(`SELECT pg_terminate_backend(pid)`) identificado vía `pg_stat_activity`.

**Corrección aplicada**: se bajó el timeout de la conexión pymssql de 60 a
45 segundos explícitamente, para que una consulta pesada falle rápido con
un mensaje claro en vez de colgar el worker de Odoo (y con él, Postgres)
indefinidamente. Si esto vuelve a pasar, revisar primero
`sys.dm_exec_requests` en SQL Server (para descartar que sea la base real)
y `pg_stat_activity` en Postgres (para encontrar y matar la transacción
colgada) antes de reintentar cualquier `docker compose exec ... odoo -u`.

## Infraestructura: pymssql

Se necesitó el driver `pymssql` para conectar Odoo (contenedor Docker
basado en `odoo:18.0`) a SQL Server. No estaba en la imagen oficial. Se
creó un `Dockerfile.odoo` propio (extiende `odoo:18.0`, instala pymssql
con `pip install --break-system-packages`) y se cambió `docker-compose.yml`
para buildear desde ese Dockerfile en vez de usar la imagen oficial
directo — así el driver persiste entre reinicios/recreaciones del
contenedor.

## Archivos relevantes

- `custom-addons/itraffic_connector/` completo (models, views, security).
- `Odoo HR/Dockerfile.odoo`, `Odoo HR/docker-compose.yml` (modificado).
- Fuente real de los SP: `D:\Repos\Softur.Serenity\Softur.Serene\Softur.Serene.Model\Migrations\DefaultDB\ScriptsBD\Sp\`.
- Contratos/tipos del webservice SOAP:
  `D:\Repos\WSBridge-Smart\Smart.ServiceContracts\SmartServiceInterfaces.Softur.cs`.

## Comandos de validación usados

- `sqlcmd -S <server> -U SA -P '***' -d itraffic_Cuenca -C -Q "..."` para
  probar cada SP de forma aislada antes de integrarlo a Odoo.
- `docker compose exec odoo odoo -u itraffic_connector -d "RRHH_CuencaDelPlata" --stop-after-init`
- Pruebas end-to-end vía `odoo shell`, con rangos de fecha acotados
  (semanas, no meses) para no repetir el incidente de conexión colgada.
