# Migrar el proyecto Odoo HR a otra PC

Fecha: 2026-09-24. Actualizado el mismo día: el proyecto pasó a estar bajo git.

## Situación previa

`D:\Repos\Odoo HR` **no estaba versionado**: tenía `.gitignore` pero nunca
se le hizo `git init`. Todo el código de los tres módulos existía en un
solo disco. El repo de GitHub que había (`Odoo_Context__Memory`) es solo
esta memoria de contexto.

Eso se corrigió: el proyecto ahora es un repo propio, y migrar es clonar y
correr un script.

## Los tamaños reales (lo que hizo viable subirlo)

| Qué | Tamaño |
|---|---|
| Carpeta del proyecto | 0,7 MB |
| Volumen `odoohr_odoo_hr_db_data` | 205,8 MB |
| **Dump de la base (`pg_dump -Fc`)** | **4,4 MB** |
| Volumen `odoohr_odoo_hr_web_data` | 17,6 MB |
| Repo completo con el dump saneado | ~2,4 MB |

El volumen de 205 MB es casi todo overhead de Postgres. El dump comprimido
entra en cualquier repo sin problema.

## Qué se decidió NO subir

### El filestore

El volumen `odoo_hr_web_data` contiene **archivos de sesión** (tokens de
login activos) y un filestore de la base vieja `odoohr`, no de
`RRHH_CuencaDelPlata`. No aporta nada y sí es riesgoso. Excluido.

### Los datos de producción de iTraffic

La base tenía 4.723 agencias y 4.789 operadores **reales de Cuenca del
Plata** (razón social, CUIT, email, dirección) y ~2.100 filas de
movimientos financieros reales. Son datos de la empresa, no del
desarrollador, y en el historial de git quedarían para siempre.

Se excluyeron, y casi no cuesta comodidad: los catálogos se rehacen con el
botón "Sincronizar catálogos" (6 s) y los resultados de los reportes
volviendo a ejecutar la consulta.

### Los secretos

En `ir_config_parameter` viven en texto plano la contraseña del SA de
iTraffic y la API key de Anthropic. **Un `pg_dump` directo se los lleva
adentro.**

## Cómo se genera el dump saneado

Sobre una **copia** de la base, nunca sobre la que está en uso:

```
docker compose up -d db
docker exec odoo_hr_db psql -U odoo -d postgres \
  -c 'DROP DATABASE IF EXISTS rrhh_export;' \
  -c 'CREATE DATABASE rrhh_export TEMPLATE "RRHH_CuencaDelPlata";'
```

Después, por stdin (ver la trampa de rutas más abajo):

```
docker exec -i odoo_hr_db psql -U odoo -d rrhh_export <<'SQL'
BEGIN;
UPDATE ir_config_parameter SET value = ''
 WHERE key IN ('hr_argentina_core.anthropic_api_key',
               'itraffic_connector.password','itraffic_connector.server',
               'itraffic_connector.user','itraffic_connector.database');
DELETE FROM ir_config_parameter WHERE key IN ('database.secret','database.uuid');
UPDATE itraffic_query SET agencia_id=NULL, operador_id=NULL,
                          state='draft', error_message=NULL;
DELETE FROM itraffic_reserva_line;      DELETE FROM itraffic_proveedor_line;
DELETE FROM itraffic_cliente_line;      DELETE FROM itraffic_caja_line;
DELETE FROM itraffic_autorizacion_line; DELETE FROM itraffic_auditoria_line;
DELETE FROM itraffic_stock_line;
DELETE FROM itraffic_agencia;           DELETE FROM itraffic_operador;
COMMIT;
SQL
docker exec odoo_hr_db pg_dump -U odoo -Fc -f /tmp/rrhh_export.dump rrhh_export
docker cp odoo_hr_db:/tmp/rrhh_export.dump "D:/Repos/Odoo HR/db/RRHH_CuencaDelPlata.dump"
docker exec odoo_hr_db psql -U odoo -d postgres -c 'DROP DATABASE rrhh_export;'
```

`DELETE` y no `TRUNCATE`: `itraffic_query` tiene FK contra los catálogos, y
`TRUNCATE` se niega por la existencia de la constraint aunque no haya
filas. `TRUNCATE ... CASCADE` se llevaría puestas las consultas guardadas.

### Verificar SIEMPRE antes de commitear

El dump `-Fc` está comprimido: `grep` sobre el binario **no prueba nada**.
Hay que descomprimirlo:

```
docker exec -i odoo_hr_db sh -c 'cat > /tmp/v.dump && pg_restore --data-only \
  --table=ir_config_parameter -f - /tmp/v.dump' < db/RRHH_CuencaDelPlata.dump \
  | grep -E 'itraffic_connector|anthropic_api_key'
```

Las filas tienen que aparecer con el valor vacío.

## Trampa: Git Bash convierte las rutas

En Git Bash, `docker exec ... -f /tmp/x.dump` falla con
`could not open output file "C:/Users/.../Temp/x.dump"`: MSYS traduce
`/tmp` a una ruta de Windows **antes** de que el comando llegue al
contenedor. Hay que anteponer `MSYS_NO_PATHCONV=1`.

Y `docker cp /tmp/archivo contenedor:/tmp/` falla igual aunque se use esa
variable, porque el origen es una ruta del host. Para meter un script al
contenedor conviene pasarlo **por stdin** (`docker exec -i ... <<'SQL'`) en
vez de `docker cp`.

## Qué hay ahora en el repo del proyecto

- Los tres módulos.
- `docker-compose.yml` con **`name: odoohr`** fijo. Antes el nombre salía
  de la carpeta: clonar con otro nombre habría creado volúmenes vacíos y
  parecería que se perdieron los datos.
- `db/RRHH_CuencaDelPlata.dump` saneado (4 MB).
- `6-Restaurar en PC nueva.bat`: levanta la base, espera a Postgres, avisa
  si la base ya existe y pide confirmación explícita antes de pisarla,
  restaura y arranca Odoo.
- `config/odoo.conf` **fuera** del repo (puede tener la clave maestra); se
  versiona `config/odoo.conf.example` y el script lo copia si falta.
- El `.gitignore` bloquea `*.dump` y `*.sql` en general, con una excepción
  explícita para el dump saneado.

## Procedimiento en la PC nueva

1. Instalar Docker Desktop.
2. Clonar el repo.
3. Correr `6-Restaurar en PC nueva.bat`.
4. Ajustes → iTraffic: cargar las credenciales del SQL Server.
5. Botón "Sincronizar catálogos".
6. `0-Crear Acceso Directo.bat`.

## Pendiente

- Anotar acá la URL del repo cuando esté creado.
- Los hashes de contraseña de los 8 usuarios viajan en el dump. Son PBKDF2
  salteados, y el repo es privado, pero conviene tenerlo presente.
- `custom-addons/hr_kpi_extended` quedó commiteado: es el módulo inicial de
  scaffolding, ya reemplazado por `hr_argentina_core`. Evaluar si se borra.
