# PRD — FTGO Platform

| Campo | Valor |
|-------|-------|
| **Producto** | FTGO (Food To Go) |
| **Versión** | 1.1 |
| **Estado** | Aprobado para diseño |
| **Horizonte** | Migración incremental 18–24 meses |
| **Origen** | [Brief §A.1–A.4], caso FTGO oficial |
| **Cambios v1.1** | Supuestos de convivencia; NFRs sin prescripción tecnológica prematura; matriz STK×C; alineación fases Strangler |

---

## 1. Contexto y objetivos

### 1.1 Contexto del negocio

FTGO opera un **marketplace de delivery de comida** que conecta consumidores, restaurantes y couriers. El sistema actual es un **monolito Java WAR** desplegado como aplicación única, con base de datos compartida y lógica de negocio altamente acoplada [Brief §A.1].

**Síntomas actuales (documentados en el caso):**

| Síntoma | Impacto en negocio | Origen |
|---------|---------------------|--------|
| Builds lentos | Menor frecuencia de releases; mayor time-to-market | [Brief §A.1] |
| Acoplamiento fuerte entre módulos | Cambios en billing afectan order taking | [Brief §A.1] |
| Escalabilidad limitada | No se escala cocina vs. checkout de forma independiente | [Brief §A.1] |
| Lock-in tecnológico | Dificultad para adoptar stacks especializados (geo, pagos) | [Brief §A.1] |
| Baja tolerancia a fallos | Caída de un proveedor externo puede bloquear pedidos | [Brief §A.1] |

### 1.2 Contexto de migración

La organización **no reemplazará el monolito de golpe**. La estrategia oficial es **Strangler Fig Pattern** durante **18–24 meses**: capacidades migradas se exponen detrás de una **fachada de enrutamiento** (API Gateway o equivalente), extrayendo bounded contexts de forma incremental mientras el **monolito legacy** atiende rutas aún no migradas [Brief §A.4] [Richardson Cap 3].

**Implicaciones para el PRD:**

- Requisitos cumplibles en **convivencia monolito + microservicios** durante todo el horizonte.
- NFR-08 (correlation IDs, tracing) es **prerrequisito de Fase 0**, no de Fase 1.
- Prioridad de extracción: capacidades con mayor presión de **carga (NFR-02)**, **disponibilidad (NFR-03)** y **resiliencia de pagos (NFR-05)** — sin asumir orden de implementación ya decidido en ADR.

### 1.3 Supuestos y restricciones (no dominio nuevo)

| ID | Supuesto / restricción | Origen |
|----|------------------------|--------|
| ASM-01 | Clientes móvil/web existentes siguen consumiendo APIs; el PRD no redefine UX de apps | [Brief §A.4] |
| ASM-02 | Durante Fase 1, **billing puede permanecer en monolito**; NFR-05 aplica igual (desacople lógico antes que físico) | [Brief §A.4] [Richardson Cap 3] |
| ASM-03 | Decisiones de **broker de eventos** y **topología sync/async** se documentan en ADR-0002, no en este PRD | Rúbrica — separación PRD/ADR |
| RES-01 | Core de microservicios nuevos en **Java/Spring Boot** | [Brief §A.4] |

### 1.4 Objetivos del producto (plataforma objetivo)

| ID | Objetivo | Métrica de éxito | Origen |
|----|----------|------------------|--------|
| OBJ-01 | Pedidos confiables en horas pico | Cumple NFR-03 | [Brief §A.4] |
| OBJ-02 | Mejorar respuesta percibida en apps | Cumple NFR-01 | [Brief §A.4] |
| OBJ-03 | Escalar por componente | Cumple NFR-06 | [Brief §A.4] [Richardson Cap 2] |
| OBJ-04 | Aislar fallos de externos (pagos) | Cumple NFR-05 | [Brief §A.4] |
| OBJ-05 | Migración sin big-bang | Cumple NFR-09 | [Brief §A.4] [Richardson Cap 3] |
| OBJ-06 | Compliance pagos y datos personales | Cumple NFR-10 | [Brief §A.4] |

### 1.5 Principios de diseño

1. **Bounded contexts = capacidades de negocio (C-01…C-07)** — no microservicios por capa técnica [Richardson Cap 2].
2. **Consistencia fuerte en aggregate Order**; eventual en reporting [Brief §A.4].
3. **Core Java/Spring Boot**; integraciones por contratos HTTP y/o mensajería (detalle en ADR) [Brief §A.4].
4. **Correlation ID** en toda interacción síncrona y asíncrona [Brief §A.4].

### 1.6 Requisitos funcionales (resumen — trazabilidad US)

| ID | Enunciado | Actor | Capacidad | Origen |
|----|-----------|-------|-----------|--------|
| RF-01 | El consumidor realiza un pedido (carrito → confirmación) | Consumidor | C-03, C-06 | [US-01] |
| RF-02 | El restaurante acepta o rechaza tickets de cocina | Restaurante | C-04 | [US-02] |
| RF-03 | El courier acepta o rechaza entregas asignadas | Courier | C-05 | [US-03] |

*Flujos derivados del brief (checkout, tracking, cancelación, menús) se especifican en `docs/FSD.md` — no duplican capacidades nuevas.*

---

## 2. Stakeholders

Lista cerrada: **seis stakeholders oficiales**. No se agregan actores adicionales.

| ID | Stakeholder | Necesidad principal | Capacidades (IDs) | Trazabilidad |
|----|-------------|---------------------|-------------------|--------------|
| STK-01 | **Consumidor** | Pedir, pagar, rastrear | C-01, C-03, C-05, C-07 | [US-01] |
| STK-02 | **Restaurante** | Gestionar menú; aceptar/rechazar/preparar tickets | C-02, C-04 | [US-02] |
| STK-03 | **Courier** | Aceptar/rechazar/reasignar entregas | C-05 | [US-03] |
| STK-04 | **Empleado FTGO (back office)** | Soporte, excepciones operativas, configuración partners | C-01–C-07 | [Brief §A.2] |
| STK-05 | **Equipo de arquitectura** | Plan de migración, NFRs, estándares observabilidad, ADRs | Transversal | [Brief §A.4] |
| STK-06 | **Sistemas externos** | Contratos de integración (pagos, geo, mensajería) | C-05, C-06, C-07 | [Brief §A.3] |

**Integraciones bajo STK-06 (caso oficial):** Stripe, Google Maps, SendGrid/Twilio.

### Matriz stakeholder × capacidad

|  | C-01 | C-02 | C-03 | C-04 | C-05 | C-06 | C-07 |
|--|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| STK-01 Consumidor | ✓ | | ✓ | | ✓ | ✓ | ✓ |
| STK-02 Restaurante | | ✓ | | ✓ | | | ✓ |
| STK-03 Courier | | | | | ✓ | | ✓ |
| STK-04 Back office | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| STK-05 Arquitectura | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| STK-06 Externos | | | | | ✓ | ✓ | ✓ |

---

## 3. Capacidades de negocio

**Siete capacidades oficiales** — definición cerrada del dominio FTGO [Brief §A.3].

### C-01: Consumer Management

| Atributo | Valor |
|----------|-------|
| **Descripción** | Registro, autenticación, perfil, direcciones, historial de pedidos |
| **Actores** | STK-01, STK-04 |
| **RF / US** | [US-01] (datos del consumidor en pedido) |
| **Servicio objetivo** | Consumer Service |
| **Migración** | Fase 2 [Brief §A.4] |

### C-02: Restaurant Management

| Atributo | Valor |
|----------|-------|
| **Descripción** | Partners, menús, horarios, disponibilidad de ítems |
| **Actores** | STK-02, STK-04 |
| **RF / US** | Derivado brief: gestión de menús |
| **Servicio objetivo** | Restaurant Service |
| **Migración** | Fase 2 |

### C-03: Order Taking

| Atributo | Valor |
|----------|-------|
| **Descripción** | Carrito, checkout, creación y validación de pedidos |
| **Actores** | STK-01 |
| **RF / US** | [US-01] |
| **Servicio objetivo** | Order Service (**aggregate Order**) |
| **Migración** | Fase 1 — prioridad [Brief §A.4] |

### C-04: Order Fulfillment / Kitchen

| Atributo | Valor |
|----------|-------|
| **Descripción** | Tickets cocina, estados de preparación, aceptación/rechazo |
| **Actores** | STK-02 |
| **RF / US** | [US-02] |
| **Servicio objetivo** | Order Service (estado); eventos hacia Restaurant (Fase 2+) |
| **Migración** | Fase 1 (estado en Order); UI kitchen puede rutear a monolito hasta Fase 2 |

### C-05: Delivery

| Atributo | Valor |
|----------|-------|
| **Descripción** | Asignación courier, tracking, reasignación, ETA |
| **Actores** | STK-01, STK-03, STK-06 (Maps) |
| **RF / US** | [US-03]; tracking [Brief §A.4] |
| **Servicio objetivo** | Delivery Service |
| **Migración** | Fase 1 — prioridad [Brief §A.4] |

### C-06: Billing & Accounting

| Atributo | Valor |
|----------|-------|
| **Descripción** | Cobro consumidor, liquidación restaurante, registros contables |
| **Actores** | STK-01, STK-04, STK-06 (Stripe) |
| **RF / US** | [US-01] + NFR-05 |
| **Servicio objetivo** | Billing Service |
| **Migración** | Fase 2 (extracción); **NFR-05 desde Fase 0** vía patrón resiliente en monolito o fachada [ASM-02] |

### C-07: Notifications

| Atributo | Valor |
|----------|-------|
| **Descripción** | Email, SMS, push por cambios de estado pedido/entrega |
| **Actores** | STK-01, STK-02, STK-03, STK-06 |
| **RF / US** | Transversal [US-01]–[US-03] |
| **Servicio objetivo** | Notification Service |
| **Migración** | Fase 2 |

---

## 4. Requisitos no funcionales

Formato obligatorio por NFR. Solo métricas del **brief** o derivadas explícitas de user stories / patrones citados.

### NFR-01: Latencia UX

- **Métrica:** **p95 < 200 ms** en operaciones síncronas críticas de UX: (a) confirmar pedido, (b) consultar estado activo del pedido, (c) aceptar ticket cocina, (d) aceptar entrega. *Excluido:* reporting, batch, webhooks asíncronos.
- **Origen:** [Brief §A.4]
- **Justificación:** Latencia en checkout y confirmación impacta conversión en marketplace. Medición en borde de API Gateway hacia cliente [Richardson Cap 3].

### NFR-02: Carga en horas pico

- **Métrica:** Soportar **5×** el tráfico baseline en almuerzo y cena sin violar NFR-01 (p95) ni NFR-03 (disponibilidad del camino crítico).
- **Origen:** [Brief §A.4]
- **Justificación:** Demanda concentrada obliga escalado independiente de componentes de pedido y entrega [Richardson Cap 2].

### NFR-03: Disponibilidad — pedidos

- **Métrica:** **99.9%** disponibilidad mensual del camino crítico: crear pedido → registro de pago (confirmado o pendiente según NFR-05) → ticket kitchen emitido.
- **Origen:** [Brief §A.4]
- **Justificación:** Ingreso core depende del flujo de pedido; requiere redundancia, health checks y despliegues sin caída coordinada del camino crítico [Richardson Cap 3].

### NFR-04: Disponibilidad — tracking

- **Métrica:** **99.5%** disponibilidad del servicio de tracking; modo degradado permitido (última posición conocida + aviso al usuario).
- **Origen:** [Brief §A.4]
- **Justificación:** Tracking mejora UX pero no debe bloquear NFR-03; SLA menor es aceptable según brief.

### NFR-05: Tolerancia a fallos — Stripe

- **Métrica:** Si Stripe no está disponible, **el pedido se crea y persiste** en estado válido; el cobro se completa de forma **asíncrona** cuando el PSP recupera. *Criterio verificable:* 0 rollbacks de pedido confirmado por timeout/5xx de Stripe en pruebas de caos [US-01].
- **Origen:** [Brief §A.4] [US-01]
- **Justificación:** El brief exige continuidad del negocio ante fallo del PSP; implementación (saga, outbox) en ADR-0002 [Richardson Cap 3].

### NFR-06: Escalabilidad independiente

- **Métrica:** Capacidad de **escalar horizontalmente** cada componente desplegado de forma independiente (sin escalar todo el monolito) para soportar NFR-02.
- **Origen:** [Brief §A.4]
- **Justificación:** Driver principal de migración a microservicios [Richardson Cap 2].

### NFR-07: Consistencia de datos

- **Métrica:** **Consistencia fuerte** en transiciones del aggregate **Order** (invariantes de estado respetadas en todo momento). **Consistencia eventual** aceptada para reporting y analítica [Brief §A.4].
- **Origen:** [Brief §A.4]
- **Justificación:** Evitar estados imposibles (p. ej. `DELIVERED` sin `PREPARED`); reporting no bloquea transacciones de pedido [Richardson Cap 5].

### NFR-08: Trazabilidad operativa

- **Métrica:** **100%** de transacciones del camino crítico incluyen **correlation ID** propagado; **distributed tracing** habilitado en todos los servicios extraídos y en la fachada Strangler.
- **Origen:** [Brief §A.4]
- **Justificación:** Diagnóstico cross-service indispensable en migración incremental [Richardson Cap 3].

### NFR-09: Migración Strangler Fig

- **Métrica:** Migración completada en **18–24 meses**; **cero** corte big-bang; cada oleada permite **revertir enrutamiento** a monolito sin redeploy completo de la plataforma.
- **Origen:** [Brief §A.4] [Richardson Cap 3]
- **Justificación:** Reduce riesgo de regresión; valida NFRs por incrementos. Detalle de rutas y flags en ADR-0001.

### NFR-10: Compliance

- **Métrica:** **PCI-DSS:** datos de tarjeta solo vía PSP (Stripe); sin almacenamiento de PAN en FTGO. **GDPR / normativa local:** capacidad de exportar y suprimir datos personales del consumidor bajo solicitud legal.
- **Origen:** [Brief §A.4]
- **Justificación:** Requisito legal; aislamiento de scope PCI al componente de billing (C-06) cuando se extraiga.

### Tabla de trazabilidad NFR → artefactos downstream

| NFR | PRD / US | FSD (previsto) | ADR | C4 |
|-----|----------|----------------|-----|-----|
| NFR-01 | [Brief §A.4] | UC-01, UC-02, UC-03 | 0002 | Container — APIs sync |
| NFR-02 | [Brief §A.4] | UC-01 | 0001 | Container — réplicas |
| NFR-03 | [Brief §A.4] | UC-01, UC-02 | 0001, 0002 | Context |
| NFR-04 | [Brief §A.4] | UC-05 | 0002 | Delivery ↔ Maps |
| NFR-05 | [US-01] | UC-01, UC-04 | 0002 | Billing ↔ Stripe |
| NFR-06 | [Brief §A.4] | Todos | 0001 | Servicios separados |
| NFR-07 | [Brief §A.4] | UC-01, UC-02 | 0002 | Order DB |
| NFR-08 | [Brief §A.4] | Transversal | 0002 | Headers / tracing |
| NFR-09 | [Brief §A.4] | — | 0001 | Monolito Legacy |
| NFR-10 | [Brief §A.4] | UC-04 | 0001, 0002 | Billing |

---

## 5. Alcance

### 5.1 Dentro de alcance (In)

| Ítem | Detalle | Origen |
|------|---------|--------|
| Migración Strangler 18–24 meses | Fachada, routing, coexistencia monolito | [Brief §A.4] [Richardson Cap 3] |
| Capacidades C-01 … C-07 | Bounded contexts objetivo | [Brief §A.3] |
| RF-01 … RF-03 / US-01 … US-03 | Flujos obligatorios del laboratorio | [US-01] [US-02] [US-03] |
| UCs derivados en FSD | Checkout, tracking, cancelación, menús | [Brief §A.4] |
| Microservicios core Java/Spring Boot | Order, Consumer, Restaurant, Delivery, Billing, Notification | [Brief §A.4] [RES-01] |
| Integraciones oficiales | Stripe, Google Maps, SendGrid/Twilio | [Brief §A.3] |
| NFR-01 … NFR-10 | Sección 4 | [Brief §A.4] |
| Mensajería / eventos de dominio | *In* como necesidad; tecnología concreta en ADR-0002 | [Richardson Cap 3] |
| Entregables arquitectura | PRD, FSD, ADR-0001/0002, C4 L1/L2 | Rúbrica laboratorio |

### 5.2 Fuera de alcance (Out)

| Ítem | Motivo | Origen |
|------|--------|--------|
| Big-bang rewrite | Contradice NFR-09 | [Brief §A.4] |
| Nuevas líneas de negocio | No en C-01…C-07 | [Brief §A.3] |
| Stakeholders adicionales | Lista cerrada de 6 | [Brief §A.2] |
| Capacidades extra (loyalty, recomendaciones ML) | Dominio no oficial | [Brief §A.3] |
| Multi-región activo-activo | No exigido en brief | [Brief §A.4] |
| Stack core distinto de Java/Spring | Brief prescribe preferencia Java | [Brief §A.4] |
| Data warehouse / BI avanzado | Solo eventual consistency base | [Brief §A.4] |
| Nuevas apps móviles | Foco backend/plataforma | Rúbrica |
| Auditoría formal PCI/GDPR en el entregable | Diseño para compliance; certificación externa posterior | [Brief §A.4] |
| Prescripción de producto concreto (Kafka, K8s, OpenTelemetry) en PRD | Evita cerrar ADR; ver ASM-03 | Mejora v1.1 |

### 5.3 Fases de entrega (alto nivel)

| Fase | Meses | Foco | Capacidades / notas |
|------|-------|------|---------------------|
| **F0** | 0–3 | Fachada Strangler, observabilidad (NFR-08), ADRs | Infra; monolito sigue 100% tráfico no piloto |
| **F1** | 4–9 | Extracción Order + Delivery | C-03, C-05; C-04 estado en Order; **C-06 lógica NFR-05 puede seguir en monolito** [ASM-02] |
| **F2** | 10–18 | Billing, Notification, Consumer, Restaurant | C-01, C-02, C-06, C-07 |
| **F3** | 19–24 | Apagado rutas monolito | Cierre NFR-09 |

### 5.4 Criterios de aceptación del PRD

- [x] 6 stakeholders oficiales + matriz STK×C
- [x] 7 capacidades con RF/US y fases coherentes
- [x] ≥ 5 NFRs (10) con métrica, origen, justificación
- [x] Alcance In/Out explícito
- [x] Strangler Fig 18–24 meses
- [x] Referencias [Brief], [US-xx], [Richardson Cap x]
- [x] Sin métricas inventadas no trazadas (v1.1)

---

## Referencias

| Referencia | Uso |
|------------|-----|
| [Brief §A.1] | Monolito, síntomas |
| [Brief §A.2] | Stakeholders |
| [Brief §A.3] | Capacidades, externos |
| [Brief §A.4] | NFRs, migración, compliance, tecnología |
| [US-01] … [US-03] | RF-01 … RF-03 |
| [Richardson Cap 2] | Decomposición, escalabilidad |
| [Richardson Cap 3] | Strangler, integración, resiliencia |
| [Richardson Cap 5] | Consistencia, sagas |

---

*Versión 1.1 — Trazabilidad: `docs/FSD.md` ← RF/NFR; `docs/adr/0001-descomposicion-microservicios.md` ← NFR-09; `docs/adr/0002-ipc-event-driven.md` ← NFR-05, NFR-07, NFR-08.*
