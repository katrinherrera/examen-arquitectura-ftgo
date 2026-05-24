# ADR-0002: Mecanismo de comunicación predominante entre servicios FTGO

| Campo | Valor |
|-------|-------|
| **Status** | Accepted |
| **Fecha** | 2026-05-24 |
| **Decisores** | Equipo de arquitectura (STK-05) |
| **Depende de** | ADR-0001 (descomposición híbrida + Strangler) [Accepted] |
| **Trazabilidad** | [PRD NFR-01] [PRD NFR-05] [PRD NFR-07] [PRD NFR-08] [Brief §A.4] [Richardson Cap 3] [Richardson Cap 5] |
| **Impacto C4** | `docs/diagrams/c4_container.mmd` **debe** incluir **Kafka / Event Broker** (`ContainerQueue`) |

---

## 1. Título y status

**ADR-0002 — Comunicación inter-servicios: modelo híbrido REST síncrono + eventos de dominio asíncronos (Kafka).**

- **Status:** Accepted
- **Supersede:** borrador `0002-event-driven-order-communication.md`

---

## 2. Contexto

Tras **ADR-0001**, FTGO evoluciona a microservicios por capability detrás de un **Strangler Fig** [PRD NFR-09]. En F1 operan **Order Service** y **Delivery Service**; en F2 **Billing**, **Notification**, **Consumer**, **Restaurant** [PRD §5.3]. El monolito legacy coexistirá y debe integrarse sin big-bang [PRD ASM-02].

**Flujos críticos (FSD):**

| UC | Necesidad de comunicación |
|----|---------------------------|
| UC-01 Tomar pedido | Escritura Order; disparo pago; ticket kitchen — latencia UX [NFR-01] |
| UC-04 Procesar pago | Integración Stripe; no revertir Order si PSP cae [NFR-05] |
| UC-02 Ticket cocina | Transición Order; notificación |
| UC-03 Asignación courier | Order ↔ Delivery; dispatch |
| UC-05 Tracking | Lectura con posible degradación [NFR-04] |

**Requisitos que condicionan IPC:**

| NFR | Implicación IPC |
|-----|-----------------|
| NFR-01 | p95 < 200 ms en confirmar pedido, aceptar ticket/entrega → favorece sync en write-path crítico |
| NFR-05 | Pedido persiste si Stripe falla → favorece async desacoplado Billing → Order |
| NFR-07 | Consistencia fuerte en aggregate Order → dueño Order; integraciones vía eventos o comandos acotados |
| NFR-08 | correlation-id en sync y async |
| NFR-09 | Strangler: publicar/consumir eventos desde monolito y MS durante dual-run |

**Problema:** ¿Cuál es el mecanismo **predominante** entre servicios (y hacia el monolito) que equilibra latencia UX, resiliencia de pagos y evolución incremental?

---

## 3. Opciones consideradas

### Leyenda de evaluación

| Símbolo | Significado |
|---------|-------------|
| ✓✓ | Muy favorable |
| ✓ | Favorable |
| ⚠ | Riesgo moderado |
| ✗ | Desfavorable |

---

### Opción 1: REST síncrono exclusivo

#### Descripción

Toda comunicación inter-servicios mediante **HTTP/REST** (JSON) con Spring WebClient/Feign. Order invoca Billing para cobro; Billing callback REST a Order; Delivery polling REST a Order. Sin broker de mensajes.

#### Pros

- Modelo simple de razonar; debugging con logs HTTP [NFR-08].
- Latencia predecible en cadenas cortas si pocos hops (NFR-01).
- Consistencia aparente “inmediata” si se usa 2PC (no recomendado) o orquestación sync.
- Menor superficie operativa (sin cluster Kafka).

#### Contras

- **Acoplamiento temporal fuerte**: Billing caído bloquea o ralentiza UC-01 si Order espera respuesta sync — viola espíritu NFR-05 salvo timeouts complejos.
- Cadenas REST largas (Order → Billing → Stripe → Order) degradan NFR-01.
- Cascadas de fallos sin buffer (NFR-03).
- Strangler dual-run frágil: monolito y MS deben exponer APIs compatibles en cada release.
- Escalar consumidores acoplado a ratio request/response.

#### Evaluación por dimensión

| Dimensión | Valoración | Notas |
|-----------|------------|-------|
| **Latencia** | ✓ en 1 hop; ⚠ en cadenas | NFR-01 en riesgo en checkout |
| **Resiliencia** | ⚠ | Circuit breakers mitigan, no eliminan acoplamiento |
| **Tolerancia a fallos** | ✗ | Stripe/ Billing down afecta path sync de Order |
| **Consistency** | ✓ local; ⚠ distribuida | Sin outbox, riesgo dual-write |
| **Escalabilidad** | ⚠ | NFR-06 parcial vía réplicas stateless |
| **Coupling** | ✗ | Conocimiento de endpoints y disponibilidad |
| **Observabilidad** | ✓ | Tracing HTTP maduro [NFR-08] |

#### Impacto en NFRs

- NFR-01 ✓ (solo si topología mínima)
- NFR-05 ✗
- NFR-07 ⚠
- NFR-08 ✓
- NFR-09 ⚠ (versionado API monolito/MS)

#### Strangler Fig

Factible con adaptadores REST en gateway, pero cada cambio de contrato exige despliegues coordinados.

---

### Opción 2: Event-driven exclusivo (Kafka)

#### Descripción

**Kafka** como único canal entre bounded contexts. Comandos y hechos de dominio vía topics (`order.events`, `payment.events`, `delivery.events`). Sin REST inter-servicios (API REST solo hacia clientes móvil/web vía gateway).

#### Pros

- **Desacoplamiento** Billing ↔ Order (NFR-05): PaymentPending consumido sin bloquear writer [Richardson Cap 3 — messaging].
- Buffer ante picos 5× (NFR-02) con consumidores escalables.
- Evolución de consumidores independiente (NFR-06).
- Strangler: monolito como consumer/producer de eventos sin API síncrona MS↔monolito.
- Publicación de hechos alimenta Notification (C-07) sin invocaciones directas.

#### Contras

- **Latencia** en confirmación de pedido: consumidor debe procesar evento antes de respuesta al cliente — NFR-01 difícil si el path es 100% async.
- **Consistencia**: solo eventual entre BC; NFR-07 requiere disciplina estricta (Order dueño, eventos de estado ordenados).
- Complejidad operativa Kafka (ops, ACLs, rebalancing).
- Debugging más duro (NFR-08 requiere trace en produce/consume).
- Riesgo de **esquemas incompatibles** y mensajes duplicados (idempotencia obligatoria).

#### Evaluación por dimensión

| Dimensión | Valoración | Notas |
|-----------|------------|-------|
| **Latencia** | ⚠ / ✗ en UX write | NFR-01 conflictivo en confirmación sync |
| **Resiliencia** | ✓✓ | Colas absorben fallos PSP |
| **Tolerancia a fallos** | ✓✓ | NFR-05 natural |
| **Consistency** | ⚠ eventual | NFR-07 vía Order aggregate + event sourcing lite |
| **Escalabilidad** | ✓✓ | Consumidores Kafka |
| **Coupling** | ✓ | Acoplamiento a contrato de evento |
| **Observabilidad** | ⚠ | Requiere headers trace en records [NFR-08] |

#### Impacto en NFRs

- NFR-01 ✗ / ⚠
- NFR-05 ✓✓
- NFR-07 ⚠
- NFR-08 ⚠ (sin estándar)
- NFR-09 ✓

#### Strangler Fig

Muy alineado si monolito publica eventos legacy; requiere **transactional outbox** en monolito y MS.

---

### Opción 3: Híbrido REST síncrono + async (Kafka) ✓

#### Descripción

- **REST sync** para operaciones **request/response** con requisito NFR-01 y lecturas interactivas:
  - Cliente → API Gateway → Order/Delivery (confirmar pedido, aceptar ticket, aceptar entrega, consultar tracking UC-05).
  - Consultas de lectura (GET estado pedido, menú proxy durante Strangler).
- **Kafka async** para **integración entre bounded contexts** y side-effects:
  - `OrderCreated`, `PaymentCaptured`, `PaymentPending`, `TicketIssued`, `OrderReady`, `DeliveryAssigned`, `OrderDelivered`.
  - Billing publica resultado de pago; Order consume; Notification consume múltiples tipos; Delivery consume `OrderReady`.
- **Patrones:** Transactional Outbox + at-least-once + consumidores idempotentes [Richardson Cap 3]; Saga coreografiada para pago (NFR-05) [Richardson Cap 5].

#### Pros

- Cumple **NFR-01** en write-path UX (respuesta HTTP tras commit local en Order).
- Cumple **NFR-05** (Billing async hacia Order).
- **NFR-07**: Order Service es sistema de registro del aggregate; eventos son derivados del commit local.
- **NFR-08**: `correlation-id` en headers HTTP y headers Kafka.
- **Strangler**: monolito publica/consume mismos topics durante F1–F2 [ADR-0001].
- **C4**: Kafka explícito como `ContainerQueue` — coherencia diagrama contenedor.

#### Contras

- **Dos modelos mentales** (REST + eventos) — curva de aprendizaje y errores de diseño.
- Riesgo **dual-write** si no hay outbox — mitigación obligatoria.
- Operación de Kafka + gateway + 6 servicios — costo medio-alto.
- **Consistencia eventual** visible entre BC (ej. notificación milisegundos después de commit).
- Ordenamiento por partición (`orderId` como key) debe diseñarse bien.

#### Evaluación por dimensión

| Dimensión | Valoración | Notas |
|-----------|------------|-------|
| **Latencia** | ✓✓ REST crítico; async fuera del path p95 |
| **Resiliencia** | ✓✓ |
| **Tolerancia a fallos** | ✓✓ NFR-05 |
| **Consistency** | ✓ Order; ⚠ entre BC |
| **Escalabilidad** | ✓✓ |
| **Coupling** | ✓ contratos evento + OpenAPI acotados |
| **Observabilidad** | ✓ con tracing unificado [NFR-08] |

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ✓ |
| NFR-02 | ✓ (Kafka buffer) |
| NFR-03 | ✓ (desacople) |
| NFR-04 | ✓ (Delivery + cache) |
| NFR-05 | ✓✓ |
| NFR-06 | ✓ |
| NFR-07 | ✓ (Order dueño) |
| NFR-08 | ✓ (propagación dual) |
| NFR-09 | ✓ (monolito como producer/consumer) |
| NFR-10 | ✓ (Billing aislado) |

#### Strangler Fig

| Fase | REST | Kafka |
|------|------|-------|
| F0 | Gateway → monolito | Opcional: outbox monolito → topics |
| F1 | Gateway → Order/Delivery; monolito adaptador | Order/Delivery publican/consumen; monolito sincroniza estado |
| F2 | + Billing, Notification APIs | Billing `Payment*` events; Notification suscriptor |
| F3 | Mismo patrón | Monolito deja de producir/consumir |

---

### Tabla comparativa consolidada

| Dimensión | REST solo | Kafka solo | Híbrido ✓ |
|-----------|-----------|------------|-----------|
| Latencia (NFR-01) | ⚠ | ✗ | ✓✓ |
| Resiliencia | ⚠ | ✓✓ | ✓✓ |
| Tolerancia fallos (NFR-05) | ✗ | ✓✓ | ✓✓ |
| Consistency (NFR-07) | ⚠ | ⚠ | ✓ |
| Escalabilidad (NFR-06) | ⚠ | ✓✓ | ✓✓ |
| Coupling | ✗ | ✓ | ✓ |
| Observabilidad (NFR-08) | ✓ | ⚠ | ✓ |
| Strangler (NFR-09) | ⚠ | ✓ | ✓✓ |
| Complejidad operativa | Baja | Alta | Media-alta |

---

## 4. Decisión

**Se adopta la Opción 3: Híbrido REST síncrono + event-driven Kafka** como mecanismo predominante de comunicación **entre** servicios (y entre servicios y monolito durante Strangler).

### Reglas de uso (normativas)

| Tipo | Mecanismo | Ejemplos FTGO |
|------|-----------|---------------|
| **Comando/consulta UX crítica** | REST sync HTTPS/JSON | UC-01 confirmar, UC-02 aceptar ticket, UC-03 aceptar entrega, UC-05 GET tracking |
| **Integración cross-BC** | Kafka (domain events) | Billing → Order (pago), Order → Delivery (`OrderReady`), * → Notification |
| **Integración externa** | REST/SDK del proveedor | Stripe, Google Maps, SendGrid/Twilio (STK-06) |
| **Strangler monolito** | REST en fachada + Kafka outbox/inbox en monolito | F0–F2 [ADR-0001] |

### Topología de eventos (v1 — C4)

| Topic (ejemplo) | Productor | Consumidores | Eventos clave |
|-----------------|-----------|--------------|---------------|
| `ftgo.order.events` | Order Service, monolito (F1) | Delivery, Notification, Billing (lectura) | `OrderCreated`, `TicketIssued`, `OrderReady`, `OrderCancelled` |
| `ftgo.payment.events` | Billing Service, monolito | Order, Notification | `PaymentCaptured`, `PaymentPending`, `PaymentFailed`, `PaymentRefunded` |
| `ftgo.delivery.events` | Delivery Service | Order, Notification | `DeliveryAssigned`, `DeliveryCompleted` |

**Particionado:** key = `orderId` para orden por pedido [NFR-07].

### Trade-offs aceptados

| Aceptamos | A cambio de |
|-----------|-------------|
| Operar Kafka + REST | NFR-05 y NFR-02 |
| Eventualidad entre BC | NFR-01 en path síncrono local |
| Outbox en cada writer | Evitar dual-write y cumplir NFR-07 |
| Contratos de evento versionados (Schema Registry recomendado) | Evolución sin romper consumidores Strangler |

**Por qué no REST solo:** no cumple NFR-05 de forma limpia; acopla Billing a latencia de Order [FSD UC-04].

**Por qué no Kafka solo:** perjudica NFR-01 en confirmación de pedido al consumidor [PRD NFR-01].

### Alineación C4 posterior (obligatoria)

El diagrama `docs/diagrams/c4_container.mmd` **debe** incluir:

- `ContainerQueue(kafka, "Event Broker", "Apache Kafka", "Eventos de dominio FTGO")`
- Relaciones `Rel(..., kafka, "Publica/Consume <Evento>", "Kafka")` entre Order, Billing, Delivery, Notification y, en F1, Monolito Legacy adaptador.

---

## 5. Consecuencias

### Positivas

- Modelo realista para marketplace: UX sync + back-office async [Richardson Cap 3].
- **NFR-05** implementable con `PaymentPending` + reintentos sin rollback de Order [FSD UC-04].
- **NFR-07** preservado con Order como dueño del aggregate.
- Notification desacoplada (C-07) vía suscripción a topics.
- Strangler: monolito sincroniza vía mismos eventos, reduciendo APIs duplicadas MS↔monolito.
- Escalado independiente de consumidores Kafka en pico (NFR-02).

### Negativas (reales)

1. **Complejidad operativa dual** (API Gateway + cluster Kafka + varios servicios) — mayor riesgo de incidentes por configuración de consumer groups.
2. **Consistencia eventual visible**: el consumidor puede ver estado retrasado (ej. notificación push tras commit Order) — soporte debe conocer el modelo.
3. **Mensajes duplicados** (at-least-once): bugs si consumidores no son idempotentes — riesgo directo a NFR-07.
4. **Debugging distribuido** más costoso pese a NFR-08 — trazas deben unir HTTP span con Kafka produce/consume.
5. **Evolución de esquemas**: cambio incompatible en evento rompe monolito Strangler y MS en paralelo — requiere gobernanza (FU-03).
6. **Costo de infraestructura** Kafka 24×7 aun en F0 si se activa outbox temprano.
7. **Riesgo de “eventos como API pública”** sin documentación — acoplamiento oculto entre equipos (anti-pattern [Richardson Cap 3]).
8. **Latencia REST inter-servicios** si se abusa de sync MS↔MS (prohibido salvo excepciones documentadas) — deriva a anti-patrón orchestration.

### Riesgos durante Strangler

- Monolito y Order Service publican eventos duplicados si no hay **única fuente de verdad** por ruta — requiere regla: solo el writer del aggregate publica `Order*`.

---

## 6. Follow-ups

| ID | Acción | Fase | Artefacto |
|----|--------|------|-----------|
| FU-01 | Definir catálogo de eventos v1 y política de versionado | F0 | Anexo ADR / AsyncAPI |
| FU-02 | Implementar transactional outbox en Order y Billing | F1 | Código + migración |
| FU-03 | Desplegar Kafka (o managed MSK/Confluent) + ACLs por topic | F0 | Infra |
| FU-04 | Propagar `correlation-id` HTTP → Kafka headers [NFR-08] | F0 | Estándar plataforma |
| FU-05 | Consumidores idempotentes + tabla `processed_events` | F1 | Order, Delivery |
| FU-06 | Actualizar **C4 Container** con `ContainerQueue` Kafka y Rel con protocolo | F0 | `c4_container.mmd` |
| FU-07 | Pruebas de caos: Stripe down + verificar NFR-05 (UC-04 GWT) | F1 | Informe QA |
| FU-08 | Anti-corruption layer monolito ↔ eventos Strangler | F1 | ADR-0001 FU-04 |
| FU-09 | Prohibir nuevas cadenas REST sync MS↔MS en checklist de PR | F1 | Guía arquitectura |

---

## Referencias

- Chris Richardson, *Microservices Patterns*, Cap 3 — IPC, messaging, transactional outbox
- Chris Richardson, *Microservices Patterns*, Cap 5 — Sagas (coreografía pago)
- `docs/PRD.md` — NFR-01, NFR-05, NFR-07, NFR-08, ASM-03
- `docs/FSD.md` — UC-01, UC-04, UC-05
- `docs/adr/0001-descomposicion-microservicios.md` — Fases F0–F3

---

*ADR-0002 Accepted — El diagrama C4 Container debe reflejar Kafka según sección 4.*
