# Odoo HR - Context Memory

Repositorio externo de memoria de contexto para el proyecto **Odoo HR**
(`D:\Repos\Odoo HR`), separado del repositorio de código.

Sigue el mismo patrón que `Codex-Context-Serenity` (memoria del proyecto
Softur.Serenity): cada conversación relevante deja una memoria fuente en
[`chats/`](./chats/), y [`PROJECT_MEMORY.md`](./PROJECT_MEMORY.md) mantiene
una síntesis consolidada y vigente.

## Por qué existe

Este proyecto acumula módulos y decisiones técnicas (RR.HH. Argentina,
conector MaxiRest, conector iTraffic) a lo largo de muchas conversaciones.
En vez de depender de que cada chat nuevo re-explique todo desde cero, se
consulta esta memoria al empezar cualquier tarea, y se actualiza al
terminar — así el contexto se acumula en vez de perderse.

## Cómo usarla

Al empezar una tarea sobre este proyecto:
1. Leer [`PROJECT_MEMORY.md`](./PROJECT_MEMORY.md) completo.
2. Revisar [`chats/`](./chats/) por nombre de archivo y leer solo las
   memorias temáticas relevantes a la tarea actual.
3. Verificar contra el código y los datos actuales cualquier dato que
   pueda haber cambiado (esta memoria son fotografías, no estado vivo).

Al terminar una tarea:
1. Crear una memoria nueva en `chats/YYYY-MM-DD-<tema>.md`.
2. Actualizar `PROJECT_MEMORY.md` solo si hay conocimiento transversal
   nuevo o cambió una conclusión consolidada.
3. Commitear y pushear.

Ver [`MEMORY_WORKFLOW.md`](./MEMORY_WORKFLOW.md) para el detalle completo
del flujo y las reglas de mantenimiento.

## Regla de seguridad

**Nunca se versionan credenciales, cadenas de conexión, tokens ni claves de
API en este repositorio.** Si una tarea requirió una credencial real
(ej. acceso a una base SQL Server de producción), la memoria describe
*qué* se hizo y *dónde* vive esa credencial en el sistema real (ej. "en
Ajustes → Conector X de Odoo"), nunca el valor en sí.
