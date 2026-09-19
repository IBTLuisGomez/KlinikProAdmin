# KlinikPro — Wiki (v1)

Referencia detallada, módulo por módulo, del funcionamiento y las reglas de negocio.

© Cytohelix Systems · Autor y fundador: Luis Fernando Mendoza Gómez.

---

## Índice
1. [Conceptos base](#1-conceptos-base)
2. [Sucursales y horarios](#2-sucursales-y-horarios)
3. [Pacientes](#3-pacientes)
4. [Catálogos: Servicios y Especialistas](#4-catálogos-servicios-y-especialistas)
5. [Agenda y disponibilidad](#5-agenda-y-disponibilidad)
6. [Caja: cobros](#6-caja-cobros)
7. [Caja: gastos](#7-caja-gastos)
8. [Caja: arqueo](#8-caja-arqueo)
9. [Finanzas](#9-finanzas)
10. [Tratamientos (solo Recuperat)](#10-tratamientos-solo-recuperat)
11. [Reportes y recibos](#11-reportes-y-recibos)
12. [Respaldo y datos](#12-respaldo-y-datos)
13. [Reglas de folios e IDs](#13-reglas-de-folios-e-ids)
14. [Preguntas frecuentes](#14-preguntas-frecuentes)

---

## 1. Conceptos base

- **Todo es por sucursal.** Cada registro pertenece a la **sucursal activa**. Al cambiar de sucursal, todos los módulos muestran los datos de esa sucursal.
- **Fuente de verdad local.** Los datos viven en IndexedDB de ese navegador. No se comparten entre equipos en v1.
- **Fecha de trabajo de Caja.** La mayoría de operaciones de Caja usan la "fecha de caja" seleccionada (por defecto, hoy en hora local).

---

## 2. Sucursales y horarios

**Ajustes → Clínica y sucursales.**

- **Sucursal activa:** selector que filtra toda la app. Botones por sucursal: **✏️ editar**, **✅ activar**, **🗑️ eliminar**.
- **Crear nueva sucursal:** botón dedicado.
- **Horario por día:** cada día (Lun–Dom) se activa/desactiva y tiene su **hora de apertura y cierre** propias (ej. L–V 9:00–20:00 y Sáb 9:00–14:00).
- **Cupos por hora** *(Recuperat):* número de pacientes simultáneos permitidos por hora (2, 3, …).
- **Logo:** en la edición Original se sube el logo para recibos; en Recuperat va embebido.

---

## 3. Pacientes

**Pestaña Pacientes.**

- **Código / ID:** 4 dígitos, **autogenerado consecutivo por sucursal** (`0001`, `0002`…), **editable** y **único** (bloquea duplicados).
- **Datos:** nombre, teléfono, email, fecha de nacimiento, notas clínicas.
- **Especialista y Tratante:** campos tipo lista+texto libre (autocompletan desde el catálogo de Ajustes; permiten valor esporádico fuera de catálogo). El **tratante** toma por defecto el especialista.
- **Banderas:** *Viene de aseguradora* y *Viene por derivación*.
- **Alta automática:** si registras un cobro con un paciente no registrado, se crea el paciente automáticamente y luego completas sus datos.
- **Gestión de tratamientos** *(Recuperat):* ver sección 10.

---

## 4. Catálogos: Servicios y Especialistas

**Ajustes → Servicios / Especialistas.** Son por sucursal.

**Servicios** (nombre, tiempo, precio) + dos atributos clave:
- **Tiempo (min):** define **cuántas horas ocupa** la cita en la agenda (50 min → 1 h; 120 min → 2 h).
- **Sesiones que otorga (paquete):** si > 0, es un paquete que concede N sesiones al venderse.
- **Cuenta como sesión de tratamiento:** si está marcado, **completar** una cita con ese servicio **descuenta 1 sesión** del plan del paciente.
  - *Sesión individual* → marcado, 0 sesiones.
  - *Paquete de N* → sesiones = N, sin marcar.
  - *Consulta* → ambos apagados.

**Especialistas** (nombre, especialidad/cédula). Alimentan los desplegables de paciente, cita y cobro.

---

## 5. Agenda y disponibilidad

**Pestaña Agenda.**

- **Cita:** paciente (enlazado por ID interno, se muestra el nombre), servicio (del catálogo, define duración), **Atendió** (por defecto el tratante del paciente), fecha y hora.
- **Enlace por ID:** la cita guarda `pacienteId`; si renombras al paciente, el cambio se refleja en todas sus citas (viejas y nuevas). Pacientes no registrados quedan como texto libre.
- **Rejilla por hora:** la agenda maneja horarios **por hora**. Cada cita ocupa las horas que dure su servicio.
- **Disponibilidad:** sección con chips de horas del día; marca ocupados y, al tocar uno libre, rellena la cita. Con **cupos por hora** (Recuperat), un horario se llena solo al alcanzar el cupo (muestra `usados/cupos`).
- **Estados:** Pendiente / Completada / Cancelada. Botón directo a WhatsApp. Al **completar**, se confirma quién atendió (y en Recuperat se consume sesión si aplica).
- **Exportar agenda:** CSV por rango de fechas.

---

## 6. Caja: cobros

**Pestaña Caja → Registrar cobro.**

- **Conceptos/productos:** uno o **varios** (concepto + importe); el **total** es la suma. Autollena precio desde el catálogo de servicios.
- **Formas de pago:**
  - *Simple:* Efectivo / Tarjeta débito / Tarjeta crédito / Transferencia. En tarjeta se captura **comisión (% o $ fijo)** y se muestra el neto. En transferencia se captura **folio de transferencia** (para conciliación).
  - *Pago mixto:* reparte el total entre varias formas; muestra **Total · Asignado · (Falta / ✓ cuadra) · Comisión · Neto** y valida que cuadre.
- **Folio:** automático **`AÑO-MES3-NNN`** (ej. `2026-AGO-001`), editable y único por sucursal.
- **Notas/observaciones:** campo opcional.
- **Acciones por cobro:** 🧾 recibo (imprimir/PDF), 📱 WhatsApp, ✏️ editar, 🗑️ eliminar.
- **Historial de cobros:** rango de fechas, imprimible.

---

## 7. Caja: gastos

**Pestaña Caja → Registrar gasto(s).**

- **Varios a la vez:** renglones adicionales; cada uno se guarda como su propio registro.
- **Tres folios por gasto:**
  - **Folio de salida (interno):** automático, único, formato **`SA-AAAA-MES-000`** (ej. `SA-2026-AGO-001`), editable.
  - **Folio ticket proveedor** (manual).
  - **Folio factura proveedor** (manual).
- **A crédito:** casilla + vencimiento → en vez de salida de caja, genera una **Cuenta por pagar (CxP)**.
- **Acciones:** 🧾 comprobante de salida, ✏️ editar, 🗑️ eliminar. **Historial de gastos** imprimible por rango.

---

## 8. Caja: arqueo

**Pestaña Caja → Arqueo.**

- **Fondo inicial:** por defecto **0**.
- Calcula efectivo esperado = fondo + ingresos en efectivo − gastos del día. Solo cuenta la **parte en efectivo** de cada cobro (respeta pagos mixtos).
- Muestra una línea informativa de **comisiones de tarjeta** del día.
- Conteo de billetes/monedas y guardado del arqueo con historial.

---

## 9. Finanzas

**Pestaña Finanzas.** Cuatro secciones:

- **Estado financiero (automático):** por rango de fechas suma **cobros** (ingresos), **gastos** y **comisiones**, y calcula la **utilidad** (ingresos − gastos − comisiones). No se captura nada.
- **Cuentas por cobrar (CxC):** libro manual. **Al marcarla Pagada** se crea automáticamente un **ingreso en Caja** (se elimina si reviertes a pendiente).
- **Cuentas por pagar (CxP):** libro manual. **Al marcarla Pagada** se crea un **gasto en Caja** (se elimina al revertir). Un gasto "a crédito" genera aquí su CxP.
- **Conciliación bancaria:** captura saldo banco/libro y partidas (depósitos/cargos); indica si cuadra o la diferencia.

---

## 10. Tratamientos (solo Recuperat)

**Pestaña Pacientes → Gestión de tratamientos.**

Un tratamiento lleva cuatro números: **Recomendadas · Pagadas · Usadas · Restantes**, con semáforo:
- 🟢 **Al corriente** · 🟡 **A revisar** (pagó menos de lo recomendado) · 🔴 **Por cobrar** (usó todas las pagadas).

**Flujo:** Consulta → asignas tratamiento (recomendadas) → al **cobrar** en Caja eliges el tratamiento y cuántas sesiones cubre (sube Pagadas) → cada **cita-sesión completada** sube Usadas → al agotar las pagadas pasa a **Por cobrar** con aviso.

- **Un tratamiento activo por paciente.**
- **Aseguradora:** los pacientes de aseguradora **no reciben paquete/promoción**.
- **💳 Generar CxC:** en tratamientos *Por cobrar / A revisar*, crea una cuenta por cobrar con el precio del paquete.

---

## 11. Reportes y recibos

- **Reporte operativo/financiero:** en Ajustes → Reportes, por rango de fechas. Se abre en **pestaña nueva** como documento membretado (KPIs, ingresos por forma/especialista, gastos, agenda, arqueos, CxC/CxP, pacientes) e imprimible a PDF.
- **Recibos de cobro y comprobantes de salida (gasto):** membretados, imprimibles y con opción de **WhatsApp**.
- **Membrete/logo:** Recuperat trae su logo embebido; Original usa el logo subido (o el logo por defecto).
- Todo documento impreso lleva el pie **© Cytohelix Systems · KlinikPro v1**.

---

## 12. Respaldo y datos

- **Respaldo `.json`:** exporta/importa todo (todas las sucursales).
- **Espejo CSV:** vincula carpeta y la app reescribe los CSV automáticamente (Chrome/Edge escritorio).
- **Borrado total:** limpia el navegador y **no toca** los CSV de la carpeta.

---

## 13. Reglas de folios e IDs

- **Código de paciente:** 4 dígitos, consecutivo por sucursal, único (bloquea duplicados).
- **Folio de cobro:** `AÑO-MES3-NNN`, consecutivo por mes/sucursal, único.
- **Folio de salida (gasto):** `SA-AAAA-MES-000`, consecutivo por mes/sucursal, único.
- Todos toman el máximo existente + 1 (no reutilizan huecos si se borra un registro).

---

## 14. Preguntas frecuentes

**¿Se comparte la información entre dos computadoras?** No en v1: cada navegador tiene su propia base. Para compartir en tiempo real se requiere backend (v2).

**¿Funciona en celular/tablet?** Sí en el navegador; la sincronización automática de CSV es solo de escritorio. Cada dispositivo tiene datos separados.

**¿El pago mixto y las comisiones?** El total viene de los conceptos; lo repartes entre formas de pago; la comisión de tarjeta se calcula sobre la parte con tarjeta y se refleja en finanzas y arqueo.

**¿Editar un cobro o gasto?** Sí, con el lápiz ✏️; actualiza el mismo registro. (En Recuperat, editar un cobro no reprocesa las sesiones de tratamiento.)

*Versión de este documento: v1.*
