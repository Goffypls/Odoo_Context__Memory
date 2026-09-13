# Organigrama de "Cuenca del Plata" cargado en Odoo

## Objetivo

Cargar empleados reales (con datos de relleno donde no se especificó) para
una demo del módulo `hr_argentina_core`, con la jerarquía real de la
empresa.

## Estado final (fotografía al 2026-09-12 — reconfirmar si cambió)

```
Patricia Duran (Dueña / Directora General) — Dirección General
├── Alejandro Galarza (Jefe de Desarrollo) — Desarrollo de Software
│   └── Mariano Antonsich (Desarrollador .NET SSr)
├── Yazu Utsugi (Gerenta Comercial) — Comercial
├── Christian — Tesorería
├── Magali — Contabilidad
├── Luz — Administración
└── Giuliana — Administración
```

- Nombre de la compañía en Odoo: **"Cuenca del Plata"**.
- Christian/Magali/Luz/Giuliana reportan directo a Patricia porque no se
  especificó un jefe de sector intermedio para Tesorería/Contabilidad/
  Administración — es un supuesto, no un dato confirmado por el usuario.
- Todos con legajo, CUIL válido (generado con el validador real del
  módulo), fecha de ingreso y obra social de relleno.
- Alejandro tiene usuario de login real (`alejandro.galarza@example.com`,
  contraseña temporal comunicada por fuera del sistema — no se guardó en
  ningún archivo). El resto son solo fichas de empleado, sin usuario.

## Nota sobre login sin servidor de correo

El entorno Docker local no tiene SMTP configurado, así que la invitación
automática por mail de Odoo no funciona. Para dar de alta un usuario que
pueda loguearse, hay que asignarle contraseña directamente al crearlo (no
dejar que Odoo mande la invitación).

## Pendiente

Ninguno identificado — es una carga de datos de demo, no una decisión
funcional abierta.
