# Contexto del Sistema: SaaS Multitenant Clínico (KlinikPro)

## 1. Visión Arquitectónica General
El sistema opera bajo una Arquitectura Hexagonal y Domain-Driven Design (DDD), enfocado en ser **FHIR-Native** desde su concepción. Toda la persistencia ocurre en PostgreSQL local, pero la exposición de datos clínicos se realiza mediante DTOs estándar FHIR R4 utilizando la librería HAPI FHIR.

- **Stack Tecnológico:** Java, Spring Boot, PostgreSQL, Docker, HAPI FHIR (R4).
- **Aislamiento Multitenant:** Estrategia basada en discriminador de columna (`tenant_id` obligatorio en toda tabla).

## 2. Patrones de Diseño de Datos
### 2.1 Modelo Híbrido Relacional/Documental
- **Datos Operativos:** Las entidades financieras, de inventario, turnos médicos y configuración de sucursales se modelan de forma relacional estricta (Normalización).
- **Datos Clínicos (FHIR-friendly):** Las notas de evolución, signos vitales y diagnósticos se almacenan en una tabla centralizada `clinical_data` utilizando una columna `JSONB` (`fhir_payload`) junto con índices relacionales para las claves foráneas (`patient_id`, `encounter_id`).

### 2.2 Exportación e Ingesta FHIR (DTOs)
- El sistema NO utiliza clientes externos ni sincronizadores hacia GCP en esta fase.
- Los controladores REST reciben y devuelven recursos puros de FHIR R4 (ej. `ca.uhn.fhir.model.dstu2.resource.Encounter`) serializados a JSON.
- `HAPI FHIR Context` se utiliza como motor de validación en la capa de aplicación para asegurar que el contenido del JSONB cumple con los perfiles estándar (SNOMED CT, LOINC) antes de persistir en PostgreSQL.

## 3. Reglas de Negocio Estrictas
1. **Fronteras de Sucursal:** Un `Location` (sucursal) delimita la agenda. El motor anti-colisiones debe validar simultáneamente: `tenant_id` + disponibilidad del `Practitioner` + disponibilidad física en el `Location` + ventana de tiempo.
2. **Inmutabilidad Clínica:** Los registros insertados en `clinical_data` no se actualizan (UPDATE). Toda modificación debe generar una nueva versión del recurso o alterar su estado (ej. de `draft` a `final`), manteniendo el historial por seguridad GxP.
3. **Desacoplamiento Financiero:** La tabla de cuentas por cobrar (`accounts_receivable`) solo guarda referencias (UUIDs) hacia el `Encounter`. Nunca duplica información de diagnósticos o tratamientos.

## 4. Roadmap de Desarrollo (Fase Actual)
- [ ] Configurar Spring Data JPA con soporte nativo para JSONB (ej. Hibernate Types).
- [ ] Construir la tabla `clinical_data` y las entidades base (`Patient`, `Practitioner`, `Location`).
- [ ] Implementar el `@RestController` de Agenda que acepte peticiones estándar y retorne objetos `Appointment` de HAPI FHIR.
- [ ] Desarrollar el motor anti-colisiones en el dominio para validación de agenda.