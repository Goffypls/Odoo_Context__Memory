# Módulo maxirest_connector

## Objetivo y alcance

Explorar la integración con "MaxiRest" (sistema de gestión gastronómica
argentina, Maxisistemas) para traer stock/insumos, ventas y cuenta
corriente de proveedores a dashboards de Odoo.

## Hallazgo clave

**MaxiRest no publica una API REST pública ni documentada para
desarrolladores externos.** Sus integraciones (PedidosYa, Rappi, Waitry,
Alax) son acuerdos de partner gestionados directamente con MaxiRest, sin
portal de desarrollador. Esto se confirmó por búsqueda web, no se asumió.

## Decisión funcional

En vez de inventar un conector real contra una API que no está documentada,
se construyó:
- Un **conector abstracto** (`maxirest.connector`, AbstractModel) que
  define el contrato: `fetch_stock()`, `fetch_sales()`,
  `fetch_supplier_invoices()`, `fetch_payment_orders()`.
- Un **conector de demostración** (`maxirest.connector.demo`) con datos de
  ejemplo (insumos gastronómicos, ventas, 2 proveedores, facturas, pagos),
  para poder probar el flujo completo de sincronización ya mismo.
- Modelos propios (`maxirest.stock.item`, `maxirest.sale`,
  `maxirest.supplier`, `maxirest.supplier.invoice`,
  `maxirest.payment.order`, `maxirest.sync.log`) — **no** se integró con
  Inventario/Compras/Facturación nativos de Odoo porque esas apps no
  estaban instaladas, y activarlas implica elegir plan de cuentas fiscal
  (decisión de negocio aparte, no algo para hacer como efecto colateral).

## Estado

Instalado y probado: sincronización corrida dos veces seguidas, sin
duplicar datos (upsert por código/ID externo). Saldos de proveedores
calculados correctamente contra los datos de ejemplo.

## Pendiente / próximos pasos

- Si en algún momento se consigue acceso real a MaxiRest (API key o
  documentación técnica de un partner), implementar `maxirest.connector.api`
  cumpliendo el mismo contrato, sin tocar el resto del módulo.
- Migrar los modelos propios a Inventario/Compras/Facturación nativos de
  Odoo cuando se decida activar esas apps en serio.

## Archivos relevantes

`custom-addons/maxirest_connector/` completo.
