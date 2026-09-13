# Flujo para alimentar la memoria del proyecto Odoo HR

## Objetivo

Cada conversación relevante aporta una memoria fuente independiente. La
memoria consolidada (`PROJECT_MEMORY.md`) se actualiza cuando aporta
conocimiento transversal; no se reescribe la historia para hacerla
coincidir con una conclusión nueva.

## Instrucción para pegar al inicio de cada conversación sobre este proyecto

```text
Usá la memoria de contexto del repositorio "Odoo HR Context"
(D:\Repos\Odoo HR Context\PROJECT_MEMORY.md) como contexto inicial, y
consultá las memorias temáticas de D:\Repos\Odoo HR Context\chats\ sólo
cuando sean relevantes para esta tarea puntual.

Durante el trabajo:
- verificá contra el código y los datos actuales (Odoo, SQL Server de
  iTraffic, etc.) cualquier información que pueda haber cambiado;
- distinguí hechos confirmados, decisiones funcionales, hipótesis y
  fotografías históricas;
- no trates conteos, saldos, IDs o estados antiguos como información
  vigente;
- si encontrás una contradicción entre esta memoria y el estado actual,
  señalala y priorizá la evidencia actual.

Al finalizar, además de resolver la tarea:
- creá una memoria nueva en
  D:\Repos\Odoo HR Context\chats\YYYY-MM-DD-<tema-descriptivo>.md, con
  objetivo, estado final, decisiones y motivos, archivos relevantes,
  cambios realizados, errores y soluciones, supuestos/riesgos,
  pendientes, comandos de validación y datos que puedan quedar
  desactualizados. No incluir razonamiento interno, credenciales ni
  transcripción del chat;
- actualizá PROJECT_MEMORY.md sólo si la tarea aporta conocimiento
  transversal o cambia una conclusión consolidada;
- hacé commit y push de los cambios en "Odoo HR Context" (nunca del
  código de "Odoo HR" — son repos separados).
```

## Regla de mantenimiento

- Una conversación relevante produce un archivo nuevo en `chats/`.
- Los archivos de `chats/` son evidencia histórica inmutable — no se
  editan retroactivamente para "corregir" una conclusión vieja; si algo
  cambió, se documenta en una memoria nueva y se marca lo anterior como
  histórico en `PROJECT_MEMORY.md`.
- `PROJECT_MEMORY.md` contiene sólo el contexto transversal y vigente,
  con advertencias de temporalidad donde corresponda.
- Nunca se versionan secretos: credenciales de bases de datos, API keys,
  tokens, contraseñas. Si una memoria necesita referirse a una
  credencial, describe dónde vive (ej. "Ajustes → iTraffic" en Odoo),
  nunca el valor.
- Si dos fuentes discrepan, se registra la discrepancia — no se elige
  silenciosamente una versión.

## Cuándo reconsolidar

Reconsolidar `PROJECT_MEMORY.md` cuando se acumulen varias memorias
nuevas sobre el mismo tema, cambie una decisión funcional central, se
agregue un módulo nuevo, o una verificación contra el sistema real
invalide una conclusión previa.
