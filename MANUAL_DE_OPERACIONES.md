# KlinikPro — Manual de operaciones (v1)

Guía práctica para el uso diario en recepción. Paso a paso, sin tecnicismos.

© Cytohelix Systems · Autor y fundador: Luis Fernando Mendoza Gómez.

---

## 0. Antes de empezar (una sola vez)

1. **Abrir la app:** doble clic en el archivo (`KlinikPro.html` o `KlinikPro_Recuperat_.html`) en Chrome o Edge.
2. **Configurar la clínica:** ve a **Ajustes → Clínica y sucursales** y captura el nombre de la clínica y de la sucursal, y el **horario por día** (activa cada día y pon su apertura/cierre).
   - *(Recuperat)* Define los **cupos por hora** (cuántos pacientes atiendes por hora).
3. **Cargar catálogos:** en **Ajustes**, agrega tus **Servicios** (nombre, tiempo, precio) y tus **Especialistas**.
4. *(Original)* **Subir logo:** en **Ajustes → Logo del recibo**.
5. **Respaldo:** activa el espejo CSV o exporta un `.json` de vez en cuando (Ajustes → Respaldo).

> **Regla de oro:** haz un **respaldo** periódico. Los datos viven en esta computadora.

---

## 1. Registrar un paciente

1. Pestaña **Pacientes**.
2. El **Código de 4 dígitos** ya viene sugerido (puedes cambiarlo; no se repite).
3. Captura nombre, teléfono, etc. Asigna **Especialista** y **Tratante** (puedes escribir uno fuera de lista si es esporádico).
4. Marca **aseguradora** o **derivación** si aplica.
5. **Guardar.**

> **Atajo:** si cobras a alguien que no está registrado, el paciente **se crea solo** al registrar el cobro; luego entras a Pacientes y completas sus datos.

---

## 2. Agendar una cita

1. Pestaña **Agenda**.
2. En **Disponibilidad**, elige la fecha: verás las **horas libres**. Toca una para que se rellene sola.
3. Escribe el **paciente** (autocompleta), el **servicio** (define cuánto dura), confirma **Atendió** y la **hora**.
4. **Confirmar cita.** Si el horario ya está lleno, la app avisa.

**Recordatorio por WhatsApp:** desde la cita, botón de WhatsApp.

**Completar una cita:** cámbiale el estado a *Completada*; confirma **quién atendió**.
*(Recuperat)* Si el servicio "cuenta como sesión", se descuenta una sesión del tratamiento del paciente.

---

## 3. Cobrar (Caja)

1. Pestaña **Caja → Registrar cobro**.
2. Captura el **paciente** y el/los **conceptos** (agrega más con "➕ Agregar concepto"). El **Total** se calcula solo.
3. Elige la **forma de pago**:
   - **Efectivo / Tarjeta / Transferencia.** En **tarjeta**, pon la **comisión** (% o $) — verás el neto. En **transferencia**, captura el **folio de transferencia**.
   - **Pago mixto:** marca la casilla y reparte el total (efectivo + tarjeta + transferencia). El resumen te dice si **cuadra**.
4. El **folio** (`AÑO-MES-000`) viene automático (editable).
5. Agrega **notas** si hace falta y **Registra el cobro**.
6. En la lista, usa **🧾** para imprimir el recibo o **📱** para enviarlo por WhatsApp.

*(Recuperat, paquetes):* al cobrar un paquete, elige el **tratamiento** del paciente y **cuántas sesiones cubre** el pago.

---

## 4. Registrar gastos (Caja)

1. Pestaña **Caja → Registrar gasto(s)**.
2. Captura concepto, proveedor y monto. Agrega más renglones si compraste varias cosas.
3. **Folios:** el **folio de salida** (`SA-AAAA-MES-000`) es automático; captura si tienes **ticket** y **factura** del proveedor.
4. Si es **a crédito** (pagas después), marca la casilla y pon el **vencimiento**: se crea una **cuenta por pagar** en Finanzas.
5. **Registrar.** Usa **🧾** para el comprobante de salida y **✏️** para editar.

---

## 5. Cierre de caja (Arqueo)

1. Pestaña **Caja → Arqueo**.
2. Captura el **fondo inicial** (por defecto 0) y cuenta el efectivo (billetes/monedas).
3. La app calcula el **efectivo esperado** y la **diferencia**. Guarda el arqueo.
   - Solo cuenta la **parte en efectivo** de los cobros; las comisiones de tarjeta salen informativas.

---

## 6. Finanzas del día/mes

1. Pestaña **Finanzas**.
2. **Estado financiero:** elige el rango de fechas y verás **ingresos, gastos, comisiones y utilidad** — se llenan solos con lo que cobraste y gastaste.
3. **Cuentas por cobrar / por pagar:** captura las deudas pendientes. Cuando las **marcas Pagadas**, se registran automáticamente como **ingreso/gasto en Caja**.
4. **Conciliación bancaria:** captura saldos y partidas para cuadrar con el banco.

---

## 7. Gestión de pacientes en tratamiento *(Recuperat)*

1. Pestaña **Pacientes → Gestión de tratamientos**.
2. Tras la consulta, **asigna el tratamiento**: elige paciente, paquete (opcional) y **sesiones recomendadas**.
3. Cada **cobro** de sesiones sube las **Pagadas**; cada **cita-sesión completada** sube las **Usadas**.
4. El **semáforo** te dice a quién revisar (pagó de menos) o **cobrar** (se le acabaron las sesiones pagadas).
5. En los que están *Por cobrar / A revisar*, el botón **💳** genera una **cuenta por cobrar**.

---

## 8. Imprimir reportes y recibos

- **Reporte del periodo:** Ajustes → Reportes → elige rango → **Generar / Exportar PDF** (se abre en pestaña nueva; usa "Guardar como PDF").
- **Historial de cobros / gastos:** desde Caja, con su rango de fechas.
- **Recibos:** desde cada cobro/gasto (🧾).
- Todos los documentos salen **membretados** y con el crédito **© Cytohelix Systems**.

---

## 9. Respaldo y buenas prácticas

- **Diario/semanal:** exporta un **`.json`** (Ajustes → Respaldo) y guárdalo en un lugar seguro (USB, nube).
- **Espejo CSV:** si trabajas en Chrome/Edge de escritorio, vincula una carpeta para tener copias en Excel siempre al día.
- **Cambio de equipo:** exporta el `.json` en el viejo e **impórtalo** en el nuevo.
- **Nunca** uses "Borrar todos los datos" salvo que estés seguro (no borra tus CSV de la carpeta, pero sí la base del navegador).

---

## 10. Problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| No veo mis datos en otra computadora | Cada equipo tiene su base | Usa respaldo `.json` para migrar (o backend en v2) |
| El PDF/recibo no abre | Bloqueador de ventanas emergentes | Permite pop-ups para el sitio/archivo |
| La sincronización CSV no funciona en el celular | Es solo de escritorio | Usa la descarga manual o respaldo `.json` |
| El pago mixto no me deja guardar | El desglose no cuadra con el total | Ajusta los montos hasta que diga "✓ cuadra" |
| "El folio/código ya existe" | Es duplicado | Usa otro; los automáticos no se repiten |

---

*Manual de operaciones · KlinikPro v1 · © Cytohelix Systems. Autor y fundador: Luis Fernando Mendoza Gómez.*
