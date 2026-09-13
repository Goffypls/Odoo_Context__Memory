# Módulo hr_argentina_core (RR.HH. Argentina)

## Objetivo y alcance

Construir un módulo único de Odoo 18 Community (`custom-addons/hr_argentina_core`
en el repo `Odoo HR`) para RR.HH. adaptado a la normativa argentina, para la
empresa "Cuenca del Plata" (base de datos Odoo `RRHH_CuencaDelPlata`, en
`D:\Repos\Odoo HR`, levantado con Docker Compose).

## Estado final

Instalado y probado end-to-end (no solo instalado — cada pieza se validó
con datos reales o simulados antes de darla por cerrada).

## Decisiones y motivos

- **No depende de `hr_payroll`**: ese módulo es Enterprise-only desde Odoo
  17, y el proyecto corre en Community. El motor de nómina es propio:
  `hr.ar.payslip` / `hr.ar.salary.rule` / `hr.ar.payslip.line`, construido
  sobre `hr_contract` (Community). Decisión tomada explícitamente por el
  usuario entre 4 opciones (motor propio, integrar OCA, migrar a
  Enterprise, o posponer nómina) — eligió motor propio.
- **Odoo 18 renombró `<tree>` a `<list>`** en vistas, y el validador de
  vistas no acepta `active_id` en contexto de botones dentro del arch (usar
  `id`). Gotchas reales encontrados al desarrollar, no asunciones.
- **Seguridad**: se reutilizan los grupos nativos de RR.HH. de Odoo en vez
  de inventar grupos nuevos — "RRHH Admin" = `hr.group_hr_manager`,
  "RRHH Manager" = `hr.group_hr_user`, "Empleado" = cualquier interno sin
  esos grupos. Reglas de registro (`ir.rule`) restringen `hr.ar.payslip` y
  `hr.ar.grievance` para que un empleado raso solo vea/edite lo propio.
  Validado creando dos empleados de prueba con usuarios reales, cada uno
  viendo solo su recibo/queja, y sin poder escribir el ajeno (`AccessError`
  confirmado). Los datos de prueba se borraron después.
- **Nota técnica de Odoo**: un empleado raso (sin `hr.group_hr_user`) no
  puede leer el modelo `hr.employee` completo por defecto — solo un
  perfil público (`hr.employee.public`). Si en el futuro se agregan
  reportes que muestren campos argentinos (CUIL, legajo) a un usuario sin
  ese grupo, hay que revisar este punto.

## Funcionalidad construida (por área)

1. **CCT** (`hr.cct`, `hr.cct.category`): convenio, categorías, salario
   básico, % presentismo, % antigüedad anual.
2. **Empleado AR** (extensión de `hr.employee`): legajo, CUIL (validado con
   algoritmo real de dígito verificador módulo 11, no solo formato),
   antigüedad calculada, CCT/categoría, obra social, sindicato,
   presentismo activo.
3. **Nómina propia**: `hr.ar.salary.rule` (reglas con condición Python +
   monto fijo/porcentaje/código Python) y `hr.ar.payslip` (recibo,
   botón Calcular que aplica las reglas en secuencia). Reglas cargadas:
   BASIC, ANTIGUEDAD, PRESENTISMO, SAC (50% de la mejor remuneración del
   semestre, Ley 23.041), JUBILACION (-11%), LEY19032 (-3%), OBRA_SOCIAL
   (-3%), SINDICATO (-2%, condicional a afiliación).
   **Pendiente**: Impuesto a las Ganancias 4ta categoría — no implementado,
   requiere un modelo de escalas con vigencia por fecha (AFIP las cambia
   varias veces al año); no inventar una fórmula fija.
4. **Quejas y denuncias** (`hr.ar.grievance`): tipo (queja/sugerencia/
   denuncia), categoría, anonimato opcional (con validación que impide
   guardar empleado visible + anónima a la vez), workflow Borrador → En
   revisión → Resuelta → Cerrada.
5. **Asistente de ayuda** (Discuss): bot con respuestas predefinidas por
   palabra clave (gratis, sin IA) + escalado opcional a Claude (Anthropic)
   si el admin lo activa en Ajustes, usando el modelo más económico
   (`claude-haiku-4-5-20251001` al momento de esta memoria). La IA se
   alimenta de una carpeta de memoria en Markdown
   (`custom-addons/hr_argentina_core/ai_memory/*.md`) que resume el estado
   real del módulo — hay que mantenerla actualizada a medida que el
   módulo cambia, para que la IA no responda con información vieja.
6. **Organigrama de "Cuenca del Plata"** cargado con datos reales de la
   empresa (ver memoria separada `2026-09-12-organigrama-cuenca-del-plata.md`).

## Archivos relevantes

- `custom-addons/hr_argentina_core/` completo (models, views, security,
  data, ai_memory).
- `custom-addons/hr_kpi_extended/`: módulo viejo, nunca llegó a
  instalarse realmente (quedó en estado `uninstalled`, sin tabla). No
  hubo nada que migrar.

## Supuestos / riesgos

- El campo `gananciaTotal` calculado en algunos contextos de RESERVA (ver
  memoria de iTraffic) puede tener valores corruptos — no es de este
  módulo, pero es una señal general de no confiar ciegamente en campos
  crudos del ERP real sin verificar.
- El chatbot con IA envía la pregunta del usuario + el contenido de
  `ai_memory/*.md` a la API de Anthropic si está activado — la clave de
  API la carga el usuario directamente en Ajustes, nunca vía código.

## Comandos de validación usados

- `docker compose exec odoo odoo -u hr_argentina_core -d "RRHH_CuencaDelPlata" --stop-after-init`
- Pruebas de seguridad y de IA vía `odoo shell -d "RRHH_CuencaDelPlata" --no-http`.
