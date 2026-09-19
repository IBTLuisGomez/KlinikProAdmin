# KlinikPro — Capa FHIR-friendly (Fase 1)
## Proyección + Import/Export de recursos FHIR R4 como DTOs

> Alcance acotado: **sin HL7v2, sin FHIR store externo, sin servidor HAPI**.
> Solo una **capa de proyección FHIR R4** sobre tu dominio Spring Boot, lista para
> escalar a interoperabilidad real en el futuro sin reescribir el núcleo.
> Producto de **Cytohelix Systems** · Autor: Luis Fernando Mendoza Gómez.

---

## 0. Decisión de alcance (qué SÍ y qué NO en Fase 1)

| Hacemos ahora | NO hacemos ahora |
|---|---|
| ✅ Modelar el dominio FHIR-friendly | ❌ HL7v2 (mensajería legacy) |
| ✅ DTOs con forma de recurso FHIR R4 | ❌ FHIR store (HAPI server / Cloud Healthcare API) |
| ✅ Mappers entidad ↔ DTO FHIR | ❌ Sincronización/doble escritura con store externo |
| ✅ Endpoints `/fhir/**` (export e import) | ❌ Suscripciones, Pub/Sub, DICOM |
| ✅ Bundles + `$everything` + `CapabilityStatement` | ❌ Lock-in a ningún proveedor |

**Idea central:** tu **PostgreSQL sigue siendo la única fuente de verdad**. FHIR es
solo una **vista de proyección** (y una puerta de entrada de import controlada). El día
que quieras un FHIR store real (HAPI self-hosted), tus DTOs y mappers ya están hechos —
solo cambia el destino.

---

## 1. Principio arquitectónico: FHIR como capa de proyección

```
                 ┌─────────────────────────────────────────┐
   Cliente FHIR  │           Controllers /fhir/**          │
   externo  ───► │   (export: read/search/$everything)     │
                 │   (import: create/update vía dominio)   │
                 └───────────────┬─────────────────────────┘
                                 │  usa
                 ┌───────────────▼─────────────────────────┐
                 │        FHIR Mappers (entity ↔ DTO)      │
                 └───────────────┬─────────────────────────┘
                     export ▲    │  import ▼ (SIEMPRE por dominio)
                 ┌───────────────┴─────────────────────────┐
                 │   Domain Services  (reglas, invariantes) │
                 └───────────────┬─────────────────────────┘
                 ┌───────────────▼─────────────────────────┐
                 │   JPA Repositories · PostgreSQL + RLS    │
                 └─────────────────────────────────────────┘
```

Reglas de oro de la capa:
1. **Export = solo lectura**, proyecta entidades a DTOs FHIR.
2. **Import NUNCA escribe directo a repos**: entra por los `Service` de dominio, así
   respeta TODAS tus invariantes (dedup, unicidad de código, multitenant).
3. La capa FHIR **no conoce SQL** ni JPA: solo habla con Services y Mappers.

---

## 2. Estructura de paquetes (Spring Boot)

```
com.cytohelix.klinikpro
├── domain/                      # tu núcleo actual (NO cambia)
│   ├── patient/  Patient (JPA), PatientService, PatientRepository
│   ├── appointment/  Appointment, AppointmentService...
│   ├── practitioner/ ...
│   └── org/  Tenant, Branch (Organization/Location en FHIR)
│
└── fhir/                        # capa NUEVA, aislada
    ├── dto/                     # records FHIR R4 (forma de recurso)
    │   ├── FhirPatient.java
    │   ├── FhirAppointment.java
    │   ├── FhirEncounter.java
    │   ├── FhirPractitioner.java
    │   ├── FhirOrganization.java
    │   ├── FhirLocation.java
    │   └── common/  Identifier, HumanName, Reference, CodeableConcept, Period, Meta
    ├── mapper/                  # PatientFhirMapper, AppointmentFhirMapper...
    ├── bundle/                  # FhirBundle, BundleBuilder (searchset / transaction)
    ├── search/                  # parseo de parámetros FHIR → filtros de dominio
    ├── controller/             # FhirPatientController (/fhir/Patient), etc.
    ├── outcome/                # OperationOutcome (errores FHIR)
    └── config/                 # FhirSystems (namespaces), CapabilityStatement, ObjectMapper FHIR
```

**Dependencia:** `fhir/` depende de `domain/`, nunca al revés. Tu dominio no sabe que
FHIR existe → puedes borrar la capa entera sin tocar el core.

---

## 3. Estrategia de identificadores (lo más crítico de todo)

FHIR distingue dos cosas que debes mapear bien:

- **`Resource.id`** → identificador **lógico/técnico** del recurso en TU sistema → tu **UUID**.
- **`Resource.identifier[]`** → identificadores **de negocio**, cada uno con un `system`
  (namespace URI) + `value`.

### Mapeo recomendado

| Concepto tuyo | FHIR | Ejemplo |
|---|---|---|
| UUID de paciente | `Patient.id` | `"id": "9f1c...uuid"` |
| Código 4 dígitos (único por sucursal) | `Patient.identifier` | system + value |
| Tenant | `Organization.id` + identifier | RFC/ID de la clínica |
| Sucursal | `Location.id` | UUID sucursal |

### Namespaces (systems) — namespacea SIEMPRE por Cytohelix + tenant

```java
public final class FhirSystems {
  public static final String BASE = "urn:cytohelix:klinikpro";

  // El código de paciente es único POR SUCURSAL → el system incluye tenant+branch
  public static String patientCode(UUID tenantId, UUID branchId) {
    return BASE + ":" + tenantId + ":" + branchId + ":patient-code";
  }
  public static String folio(UUID tenantId, UUID branchId) {
    return BASE + ":" + tenantId + ":" + branchId + ":folio";
  }
}
```

**Por qué importa:** tu código de paciente NO es único a nivel global (se repite entre
sucursales). Si lo expones como identifier sin namespacear por sucursal, un sistema
externo creería que dos pacientes distintos son el mismo. El `system` resuelve la
colisión sin cambiar tu esquema.

---

## 4. Multitenant en la capa FHIR (sin fugas)

- Todos los endpoints `/fhir/**` viven **detrás del `TenantFilter` + JWT** que ya tienes.
- El `TenantContext` (ThreadLocal) acota cada query → los mappers solo ven datos del
  tenant activo. **RLS de PostgreSQL sigue siendo el blindaje de fondo.**
- El tenant se refleja en FHIR como `Organization`; cada recurso lleva su referencia:
  - `Patient.managingOrganization → Organization/{tenantId}`
  - `Location.managingOrganization → Organization/{tenantId}`
  - `Appointment` → participantes referencian `Patient`, `Practitioner`, `Location`.
- **Invariante:** un export jamás incluye recursos de otro tenant; `$everything` de un
  paciente solo trae recursos de ese mismo paciente y tenant.

---

## 5. Mapeo entidad → recurso FHIR R4

| Entidad KlinikPro | Recurso FHIR R4 | Estado Fase 1 |
|---|---|---|
| Paciente | `Patient` | ✅ Core |
| Especialista / tratante | `Practitioner` (+ `PractitionerRole`) | ✅ Core |
| Sucursal | `Location` | ✅ Core |
| Tenant / clínica | `Organization` | ✅ Core |
| Servicio | `HealthcareService` | ✅ Core |
| Cita | `Appointment` | ✅ Core |
| Cita completada / sesión | `Encounter` | ✅ Core |
| Plan de tratamiento | `CarePlan` | 🟡 Extensión |
| Sesión ejecutada | `Procedure` | 🟡 Extensión |
| Cobro | `Invoice` / `ChargeItem` | 🟡 Extensión (financiero) |

**Regla de negocio:** el módulo financiero (POS, caja, CxC/CxP) **NO se expone como
FHIR en Fase 1** salvo un `Invoice` mínimo opcional. FHIR es para el expediente
clínico e interoperabilidad, no para tu contabilidad interna.

---

## 6. Los DTOs — records Java 17 con forma FHIR R4

Records inmutables, serialización directa a JSON FHIR con Jackson. `@JsonInclude` para
omitir nulos (FHIR no serializa campos vacíos).

### 6.1 Tipos comunes

```java
public record Identifier(String use, String system, String value) {}

public record HumanName(String use, String text, String family, List<String> given) {}

public record Reference(String reference, String display) {} // "Patient/uuid"

public record Coding(String system, String code, String display) {}
public record CodeableConcept(List<Coding> coding, String text) {}

public record Period(OffsetDateTime start, OffsetDateTime end) {}

public record Meta(String versionId, OffsetDateTime lastUpdated, List<String> profile) {}

public record ContactPoint(String system, String value, String use) {} // phone/email
```

### 6.2 `FhirPatient` (plantilla completa — el resto sigue el mismo patrón)

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record FhirPatient(
    String resourceType,          // siempre "Patient"
    String id,                    // UUID del paciente
    Meta meta,
    List<Identifier> identifier,  // código 4 dígitos namespaceado
    Boolean active,               // = !bajaLogica
    List<HumanName> name,
    List<ContactPoint> telecom,   // teléfono
    Reference managingOrganization, // Organization/{tenantId}
    List<Reference> generalPractitioner // Practitioner/{tratanteId}
) {
    public static FhirPatient of(...) { /* construido por el mapper */ }
}
```

### 6.3 `FhirAppointment`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record FhirAppointment(
    String resourceType,   // "Appointment"
    String id,
    Meta meta,
    String status,         // proposed|booked|arrived|fulfilled|cancelled|noshow
    List<CodeableConcept> serviceType, // = tu Servicio
    OffsetDateTime start,
    OffsetDateTime end,    // start + duración del servicio
    Integer minutesDuration,
    List<AppointmentParticipant> participant // paciente, practitioner, location
) {}

public record AppointmentParticipant(Reference actor, String status) {}
```

Mapeo de estados (tu máquina de estados → FHIR):

| Tu estado | `Appointment.status` |
|---|---|
| AGENDADA | `booked` |
| CONFIRMADA | `arrived` |
| COMPLETADA | `fulfilled` |
| CANCELADA | `cancelled` |
| NO_ASISTIO | `noshow` |

### 6.4 `FhirEncounter` (cuando la cita se completa / consume sesión)

```java
public record FhirEncounter(
    String resourceType,   // "Encounter"
    String id,
    String status,         // "finished"
    Reference subject,     // Patient/{id}
    List<Reference> participant, // Practitioner que atendió
    Period period,
    Reference appointment, // Appointment/{id} que lo originó
    Reference serviceProvider // Organization/{tenantId}
) {}
```

---

## 7. Mappers (entidad ↔ DTO)

Un mapper por recurso. Sin lógica de negocio, solo traducción. Export puro; import
devuelve un **comando de dominio** (no una entidad), para que el Service aplique reglas.

```java
@Component
public class PatientFhirMapper {

  // EXPORT: entidad → DTO FHIR
  public FhirPatient toFhir(Patient p) {
    var idSystem = FhirSystems.patientCode(p.getTenantId(), p.getBranchId());
    return new FhirPatient(
      "Patient",
      p.getId().toString(),
      new Meta(String.valueOf(p.getVersion()), p.getUpdatedAt(),
               List.of(FhirSystems.BASE + "/StructureDefinition/klinikpro-patient")),
      List.of(new Identifier("official", idSystem, p.getCodigo())),
      !p.isBajaLogica(),
      List.of(new HumanName("official", p.getNombre(), null, null)),
      p.getTelefono() == null ? null
        : List.of(new ContactPoint("phone", p.getTelefono(), "mobile")),
      new Reference("Organization/" + p.getTenantId(), null),
      p.getTratanteId() == null ? null
        : List.of(new Reference("Practitioner/" + p.getTratanteId(), null))
    );
  }

  // IMPORT: DTO FHIR → comando de dominio (el Service decide qué hacer)
  public UpsertPatientCommand toCommand(FhirPatient r) {
    String codigo = r.identifier() == null ? null
        : r.identifier().stream().findFirst().map(Identifier::value).orElse(null);
    String nombre = r.name() == null ? null
        : r.name().stream().findFirst().map(HumanName::text).orElse(null);
    String tel = r.telecom() == null ? null
        : r.telecom().stream()
            .filter(c -> "phone".equals(c.system())).findFirst()
            .map(ContactPoint::value).orElse(null);
    return new UpsertPatientCommand(codigo, nombre, tel /*, tratante, etc.*/);
  }
}
```

---

## 8. Controllers / endpoints FHIR

Rutas estilo FHIR bajo `/fhir`. `Content-Type: application/fhir+json`.

```java
@RestController
@RequestMapping("/fhir/Patient")
public class FhirPatientController {

  private final PatientService service;      // dominio
  private final PatientFhirMapper mapper;
  private final BundleBuilder bundles;

  // READ  ── GET /fhir/Patient/{id}
  @GetMapping("/{id}")
  public FhirPatient read(@PathVariable UUID id) {
    return mapper.toFhir(service.getByIdOrThrow(id)); // Service acota por tenant
  }

  // SEARCH ── GET /fhir/Patient?identifier=...&name=...&active=true
  @GetMapping
  public FhirBundle search(@RequestParam Map<String,String> params) {
    var criteria = PatientSearch.parse(params);        // capa search/
    var page = service.search(criteria);
    return bundles.searchset("Patient",
        page.map(mapper::toFhir).toList(), page.getTotalElements());
  }

  // $everything ── GET /fhir/Patient/{id}/$everything
  @GetMapping("/{id}/$everything")
  public FhirBundle everything(@PathVariable UUID id) {
    var p = service.getByIdOrThrow(id);
    return bundles.everythingForPatient(p); // Patient + Appointments + Encounters + CarePlan
  }

  // IMPORT create ── POST /fhir/Patient   (SIEMPRE por dominio)
  @PostMapping
  public ResponseEntity<FhirPatient> create(@RequestBody FhirPatient body) {
    var cmd = mapper.toCommand(body);
    var saved = service.upsertFromFhir(cmd); // aplica dedup + unicidad + tenant
    return ResponseEntity.status(HttpStatus.CREATED).body(mapper.toFhir(saved));
  }
}
```

---

## 9. Bundles y `$everything`

- **searchset** → resultado de búsqueda (paginado, con `total`).
- **transaction** → import de varios recursos atómico (todo o nada, `@Transactional`).
- **`$everything`** → expediente completo de un paciente en un solo Bundle.

```java
public record FhirBundle(
    String resourceType,   // "Bundle"
    String type,           // "searchset" | "transaction" | "collection"
    Integer total,
    List<BundleEntry> entry
) {}

public record BundleEntry(String fullUrl, Object resource) {}
```

`everythingForPatient(p)` ensambla, respetando aislamiento por tenant:
`Patient` + sus `Appointment[]` + `Encounter[]` + (si aplica) `CarePlan` + `Procedure[]`.

---

## 10. Import: reglas de negocio (la parte delicada)

El import **no** es un `INSERT` ciego. `service.upsertFromFhir(cmd)` aplica, en orden:

1. **Aislamiento tenant** — el recurso se crea SOLO en el tenant del JWT; se ignora
   cualquier `managingOrganization` entrante que apunte a otro tenant (o se rechaza con
   `OperationOutcome`).
2. **Resolución por identifier** — si llega un código de paciente existente en esa
   sucursal → es UPDATE; si no → CREATE.
3. **Deduplicación** — misma regla que el alta manual: choque por nombre/teléfono →
   `OperationOutcome` con severidad `warning` y referencia al candidato, sin crear ciego.
4. **Unicidad de código** — si el código namespaceado ya existe para otro UUID → rechazo.
5. **Autollenado** — tratante = especialista si no viene; código autogenerado si viene vacío.
6. **Idempotencia** — un `identifier` + `meta.versionId` repetido no duplica.

**Invariante clave:** *"FHIR es la entrada, tus invariantes mandan."* Nunca un import
puede dejar el dominio en un estado que el alta manual no permitiría.

---

## 11. CapabilityStatement y OperationOutcome

### 11.1 `GET /fhir/metadata` → CapabilityStatement

Le dice a cualquier sistema externo qué hablas. Declara recursos + interacciones
soportadas (`read`, `search-type`, `create`, `$everything`) y tu `fhirVersion: "4.0.1"`.
Sin esto, no eres "descubrible" como servidor FHIR.

### 11.2 Errores → `OperationOutcome` (nunca stacktraces)

```java
public record OperationOutcome(
    String resourceType,        // "OperationOutcome"
    List<OperationOutcomeIssue> issue
) {}
public record OperationOutcomeIssue(
    String severity,            // error | warning | information
    String code,                // e.g. "duplicate", "invalid", "not-found"
    String diagnostics
) {}
```

Un `@RestControllerAdvice` traduce tus excepciones de dominio a `OperationOutcome` con
el HTTP status correcto (404, 409 en duplicado, 422 en validación).

---

## 12. Qué recursos entran en Fase 1

**Core (implementar ya):** `Patient`, `Practitioner`, `PractitionerRole`,
`Organization`, `Location`, `HealthcareService`, `Appointment`, `Encounter`.

**Extensión (cuando el core esté sólido):** `CarePlan` (planes de tratamiento),
`Procedure` (sesiones ejecutadas), `Invoice` mínimo.

**Fuera de Fase 1:** todo lo financiero interno, HL7v2, DICOM, Subscriptions.

---

## 13. Reglas de negocio / invariantes de la capa FHIR

1. **Proyección de solo lectura por defecto**; el import se habilita por endpoint y
   siempre pasa por Services de dominio.
2. **Aislamiento tenant heredado** del `TenantContext`; RLS como blindaje de fondo.
3. **`identifier.system` namespaceado** por Cytohelix + tenant (+ sucursal donde aplique).
4. **`Resource.id` = UUID**; nunca exponer PKs internas secuenciales.
5. **`meta.versionId` / `lastUpdated`** mapeados desde tu `version` / `updated_at` (optimistic locking).
6. **El módulo financiero no se expone** salvo `Invoice` mínimo opcional.
7. **`fhir/` no depende de JPA**; el dominio no depende de `fhir/` → capa desechable/reemplazable.
8. **Conformidad declarada** vía `CapabilityStatement`; errores vía `OperationOutcome`.
9. **Fecha/hora en offset UTC** en el borde FHIR (`OffsetDateTime`), aunque el dominio use hora local.

---

## 14. Ruta de escalamiento (cuando se retome, NO ahora)

Como los DTOs, mappers y endpoints ya existen, el salto a interoperabilidad real es
incremental y sin reescribir dominio:

1. **Hoy (Fase 1):** proyección FHIR R4 in-process (este documento). ✅
2. **Después:** cambiar el destino de export/import a **HAPI FHIR self-hosted**
   (reusa Postgres + RLS, sin lock-in). Los mappers no cambian; solo el sink.
3. **Bajo demanda de un cliente:** HL7v2 store para hablar con labs/HIS; DICOM para imagen.

Nada de la Fase 1 se tira: es el cimiento exacto de las fases siguientes.

---

*Capa FHIR-friendly diseñada como proyección desechable sobre tu dominio: interoperable
desde ya, sin lock-in y sin comprometer tu núcleo transaccional multitenant.*
