# KlinikPro — v1

**Sistema integral de gestión para clínicas** (pacientes, agenda, caja, finanzas y reportes) en un **único archivo HTML**, sin servidor y con funcionamiento **offline**.

© **Cytohelix Systems** — Todos los derechos reservados.
Autor y fundador: **Luis Fernando Mendoza Gómez**.

---

## 1. Qué es

KlinikPro es una aplicación de escritorio/navegador de **archivo único** (`.html`) que corre localmente sin instalación ni backend. Toda la lógica, el estilo y los datos viven en el mismo archivo y en la base de datos del navegador. Está pensada para la recepción de una clínica: registrar pacientes, agendar citas, cobrar, controlar gastos y llevar finanzas básicas.

### Ediciones (v1)

| Edición | Archivo | Base de datos | Diferencia |
|---|---|---|---|
| **Original** (producto a la venta) | `KlinikPro.html` | IndexedDB v3 | Base común, con carga de logo propio para recibos |
| **Recuperat** (instancia de clínica) | `KlinikPro_Recuperat_.html` | IndexedDB v4 | Todo lo anterior **+ módulo de Tratamientos/sesiones** y **cupos por hora**; logo Recuperat embebido |

Ambas ediciones comparten el 95% del código; la edición Recuperat es un superconjunto.

---

## 2. Tecnología

- **Frontend:** HTML + CSS + JavaScript *vanilla* (sin frameworks ni dependencias externas).
- **Persistencia:** **IndexedDB** (base de datos del navegador). La app es la fuente de verdad local; no requiere internet para operar.
- **Respaldo:** exportación/importación en `.json` y **espejo automático a CSV** mediante la **File System Access API** (solo Chrome/Edge de escritorio).
- **Impresión/PDF:** se genera un documento membretado en pestaña nueva y se usa la **impresión nativa del navegador** ("Guardar como PDF"). Sin librerías externas.
- **Diseño:** sistema de diseño propio ("Clinical Precision") con tokens de color, tipografía Inter y componentes en CSS.

No hay proceso de build. El archivo se abre directamente.

---

## 3. Cómo ejecutar

1. Copia el archivo de tu edición (`KlinikPro.html` o `KlinikPro_Recuperat_.html`) al equipo de recepción.
2. Ábrelo con **doble clic** en Chrome, Edge o Safari. No requiere instalación ni internet.
3. (Recomendado para producción) **Hospédalo en una URL** (hosting estático) y, en el celular/tablet, usa "Agregar a pantalla de inicio".

> **Importante:** los datos viven en el **navegador de ese equipo**. Cada equipo/navegador tiene su propia base de datos aislada; no se comparten datos entre dispositivos en esta versión (ver *Roadmap*).

---

## 4. Módulos

- **Inicio (Dashboard):** métricas del día y próximas citas.
- **Pacientes:** expediente, código/ID de 4 dígitos, especialista/tratante, banderas (aseguradora, derivación).
- **Agenda:** citas por hora, disponibilidad de horarios, exportación por rango.
- **Caja:** cobros (con productos múltiples, pago mixto, comisiones, recibo/WhatsApp) y gastos (multi-línea, 3 folios, a crédito), arqueo diario.
- **Finanzas:** estado financiero automático, cuentas por cobrar/pagar, conciliación bancaria.
- **Ajustes:** clínica/sucursales, horarios, servicios, especialistas, reportes, respaldo, logo (original) y "Acerca de".
- **Tratamientos** *(solo Recuperat):* planes de sesiones por paciente con disparadores de cobranza.

---

## 5. Modelo de datos (IndexedDB)

Base de datos `KlinikProDB`. Object stores:

| Store | Contenido |
|---|---|
| `patients` | Pacientes (código, nombre, contacto, especialista/tratante, banderas) |
| `appointments` | Citas (pacienteId, servicioId, atendio, fecha/hora, estado) |
| `transactions` | Cobros (items[], pagos[], comisión, folio, notas) |
| `expenses` | Gastos (folioSalida SA, ticket/factura proveedor) |
| `cashcounts` | Arqueos de caja |
| `receivables` | Cuentas por cobrar (CxC) |
| `payables` | Cuentas por pagar (CxP) |
| `bankmovs` | Partidas de conciliación bancaria |
| `services` | Catálogo de servicios/paquetes (tiempo, precio, sesiones, esSesion) |
| `specialists` | Catálogo de especialistas |
| `treatments` | *(solo Recuperat)* Planes de tratamiento/sesiones |
| `branches` | Sucursales (clínica, horarios por día, cupos, logo) |
| `settings` | Configuración (sucursal activa, vínculo de carpeta CSV) |

Todos los registros de negocio llevan `sucursalId` (multi-sucursal). Este modelo se traduce de forma directa a tablas SQL para una futura migración a backend.

---

## 6. Arquitectura interna

- **Capa de datos:** ~5 funciones (`dbGetAll`, `dbAdd`, `dbPut`, `dbDelete`, `dbGet`) + `reloadCache()`. Toda la app accede a los datos por aquí; migrar a una API/servidor implica reescribir **solo estas funciones**.
- **Caché en memoria:** `allData` (todo) y `cache` (filtrado por sucursal activa). El filtrado por sucursal se aplica en `applyBranchFilter()`.
- **Render:** funciones `render*` por módulo; `renderAll()` orquesta.
- **Migraciones:** al subir `DB_VERSION`, se crean stores nuevos y se rellenan campos faltantes (p. ej. `sucursalId`, enlaces de cita→paciente).

---

## 7. Respaldo y datos

- **Exportar/Importar `.json`:** respaldo completo (todas las sucursales) desde Ajustes.
- **Espejo CSV:** vincula una carpeta local y la app reescribe los CSV al registrar cambios (solo Chrome/Edge escritorio; en móvil solo descarga manual).
- **Borrado total:** limpia los datos del navegador y **conserva los CSV de la carpeta**.

---

## 8. Autoría y licencia

Software propiedad de **Cytohelix Systems**, del que **Luis Fernando Mendoza Gómez** es dueño y fundador. Todos los derechos reservados. El crédito de autoría aparece en la sección "Acerca de" y en el pie de todo documento impreso (recibos y reportes).

---

## 9. Roadmap (v2 en adelante)

- Backend + base de datos real (multi-dispositivo/multi-sucursal compartido en tiempo real).
- Roles y autenticación (recepción vs. especialista).
- Sincronización offline-first.
- Automatizaciones adicionales de finanzas y reportes.

*Versión de este documento: v1.*
# KlinikProAdmin
