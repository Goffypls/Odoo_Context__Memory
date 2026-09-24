# Migrar el proyecto Odoo HR a otra PC

Fecha: 2026-09-24.

## Lo primero: el repo de GitHub NO alcanza

`Odoo_Context__Memory` es solo la memoria de contexto. **El proyecto en sí
(`D:\Repos\Odoo HR`) no está bajo control de versiones**: tiene un
`.gitignore` pero nunca se le hizo `git init`. El código de
`hr_argentina_core`, `maxirest_connector` e `itraffic_connector` vive
únicamente en el disco de esa máquina.

Mientras eso siga así, migrar = copiar archivos a mano. La solución de
fondo es poner la carpeta en un repo privado; ahí sí "clonar y listo".

## Qué hay que llevarse

| Qué | Dónde está | Tamaño | ¿Por qué? |
|---|---|---|---|
| La carpeta del proyecto | `D:\Repos\Odoo HR` | ~700 KB | Los tres módulos, `docker-compose.yml`, `Dockerfile.odoo`, los `.bat`, `config/odoo.conf`. No está en git. |
| La base de datos | volumen Docker `odoohr_odoo_hr_db_data` | cientos de MB | `RRHH_CuencaDelPlata`: empleados, organigrama, consultas guardadas, catálogos sincronizados (~9.500 filas) y **las credenciales en `ir_config_parameter`**. |
| El filestore | volumen Docker `odoohr_odoo_hr_web_data` | variable | Adjuntos y archivos subidos a Odoo. |
| `.wslconfig` | `C:\Users\<usuario>\.wslconfig` | 1 KB | Opcional. Tope de RAM de WSL2. Se puede reescribir. |

El prefijo `odoohr_` de los volúmenes sale del nombre de proyecto que
Compose deriva de la carpeta (se vio en la red `odoohr_default`).
**Confirmarlo con `docker volume ls` antes de restaurar**: si el nombre no
coincide, Docker crea un volumen vacío y parece que se perdieron los datos.

## Qué NO hace falta llevarse

Todo esto se baja o se reconstruye solo en la PC nueva:

- **Docker Desktop**: se instala desde la web (o con el
  `1-Instalar-Docker.bat` del propio proyecto).
- **Las imágenes `postgres:15` y `odoo:18.0`**: las baja
  `docker compose up -d`.
- **La imagen propia de Odoo con `pymssql`**: la buildea Compose sola a
  partir de `Dockerfile.odoo`.
- **El repo de memoria**: se clona de GitHub.

## Procedimiento

### En la PC vieja

```
cd "D:/Repos/Odoo HR"
docker compose up -d db
docker volume ls                # confirmar los nombres reales
docker exec odoo_hr_db pg_dump -U odoo -Fc -f /tmp/rrhh.dump RRHH_CuencaDelPlata
docker cp odoo_hr_db:/tmp/rrhh.dump "D:/backup-odoo/rrhh.dump"
docker run --rm -v odoohr_odoo_hr_web_data:/data -v "D:/backup-odoo:/backup" \
  alpine tar czf /backup/filestore.tar.gz -C /data .
```

Se usa `docker cp` en vez de redirigir la salida de `pg_dump`: mandar un
dump binario por stdout desde Git Bash en Windows lo corrompe.

Después, copiar a un USB: la carpeta `D:\Repos\Odoo HR` completa y
`D:\backup-odoo\`.

### En la PC nueva

1. Instalar Docker Desktop y dejarlo arrancar una vez.
2. Copiar la carpeta del proyecto (misma ruta o cualquier otra).
3. Levantar solo la base y restaurar:

```
cd "<ruta>/Odoo HR"
docker compose up -d db
docker cp "<ruta>/backup-odoo/rrhh.dump" odoo_hr_db:/tmp/rrhh.dump
docker exec odoo_hr_db createdb -U odoo RRHH_CuencaDelPlata
docker exec odoo_hr_db pg_restore -U odoo -d RRHH_CuencaDelPlata /tmp/rrhh.dump
docker run --rm -v odoohr_odoo_hr_web_data:/data -v "<ruta>/backup-odoo:/backup" \
  alpine tar xzf /backup/filestore.tar.gz -C /data
docker compose up -d
```

4. Correr `0-Crear Acceso Directo.bat` para rehacer el acceso directo.

**Ojo con la ruta**: si la carpeta se llama distinto, Compose usa otro
prefijo de proyecto y los volúmenes cambian de nombre. Conviene mantener el
mismo nombre de carpeta, o fijarlo con `name: odoohr` en el
`docker-compose.yml`.

## Seguridad: el dump lleva credenciales adentro

El dump de Postgres incluye la tabla `ir_config_parameter`, y ahí viven en
texto plano la contraseña del SQL Server de iTraffic y la API key de
Anthropic si está cargada.

- **Nunca subir el `.dump` a GitHub**, ni siquiera a un repo privado.
- Pasarlo por USB o canal privado, y borrarlo del USB al terminar.
- Cuando el proyecto se ponga bajo git, el `.gitignore` tiene que excluir
  explícitamente `*.dump`, `*.sql`, `*.tar.gz` y `config/odoo.conf`.

Además `config/odoo.conf` tiene el `admin_passwd` todavía en el valor de
ejemplo (`admin_master_password_cambiar`) — buen momento para cambiarlo.

## Pendiente

Poner `D:\Repos\Odoo HR` en un repo privado de GitHub. Es el arreglo de
fondo: hoy una sola falla de disco se lleva puesto todo el código.
