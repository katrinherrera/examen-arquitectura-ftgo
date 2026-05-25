# ADR-0002: Mecanismo de comunicación predominante entre servicios FTGO

| Campo | Valor |
|-------|-------|
| **Status** | Accepted |
| **Versión** | 1.1 |
| **Fecha** | 2026-05-24 |
| **Decisores** | Equipo de arquitectura (STK-05) |
| **Depende de** | ADR-0001 v1.1 [Accepted] |
| **Trazabilidad** | [PRD NFR-01] [PRD NFR-05] [PRD NFR-07] [PRD NFR-08] [Richardson Cap 3] [Richardson Cap 5] |
| **Impacto C4** | `c4_container.mmd` — **ContainerQueue** Kafka obligatorio |
| **Cambios v1.1** | F0 Kafka infra alineado ADR-0001; tablas NFR completas; regla writer único; matriz coherencia |

---

## 1. Título y status

**ADR-0002 — IPC híbrida: REST síncrono (cliente y UX) + Kafka async (integración entre bounded contexts).**

- **Status:** Accepted
- **Predominancia:** **Async (Kafka)** entre microservicios y monolito-adaptador; **REST** hacia clientes y operaciones UX críticas [PRD NFR-01].

---

## 2. Contexto

Post **ADR-0001** (híbrido incremental + Strangler). F1: Order + Delivery; billing puede permanecer en **monolito** con eventos [PRD ASM-02].

| UC [FSD] | IPC necesaria |
|----------|---------------|
| UC-01 | REST confirmar; evento post-commit; pago vía UC-04 |
| UC-04 | Stripe sync; resultado → Kafka `payment.events` |
| UC-02, UC-03 | REST aceptar ticket/entrega [NFR-01] |
| UC-05 | REST lectura tracking [NFR-04] |

**Problema:** ¿Mecanismo predominante entre BC que cumpla NFR-01, NFR-05, NFR-07 y Strangler?

---

## 3. Opciones consideradas

### Opción 1: REST síncrono exclusivo (inter-servicios)

#### Descripción

HTTP/JSON entre Order, Billing, Delivery, Notification. Cliente → gateway → servicios. Sin broker.

#### Pros / Contras

- ✓ Simplicidad ops; ✓ tracing HTTP.
- ✗ NFR-05; ✗ cadenas largas NFR-01; ✗ coupling temporal; ✗ Strangler frágil (versionado API dual).

#### Evaluación dimensional

| Dimensión | Valoración |
|-----------|------------|
| Latencia | ⚠ |
| Resiliencia | ⚠ |
| Tolerancia fallos (NFR-05) | ✗ |
| Consistency (NFR-07) | ⚠ dual-write |
| Escalabilidad (NFR-06) | ⚠ |
| Coupling | ✗ |
| Observabilidad (NFR-08) | ✓ |

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ⚠ |
| NFR-05 | ✗ |
| NFR-07 | ⚠ |
| NFR-08 | ✓ |
| NFR-09 | ⚠ |

#### Complejidad operativa / Migración

Baja ops; migración Strangler requiere **APIs espejo** monolito↔MS por cada release.

---

### Opción 2: Event-driven exclusivo (Kafka inter-servicios)

#### Descripción

Solo Kafka entre BC; REST únicamente cliente → gateway. Confirmación pedido vía consume evento.

#### Pros / Contras

- ✓ NFR-05, NFR-02 buffer; ✓ Strangler por topics.
- ✗ NFR-01 en write UX; ✗ ops Kafka alta; ✗ eventualidad visible.

#### Evaluación dimensional

| Dimensión | Valoración |
|-----------|------------|
| Latencia | ✗ / ⚠ UX |
| Resiliencia | ✓✓ |
| Tolerancia fallos | ✓✓ |
| Consistency | ⚠ |
| Escalabilidad | ✓✓ |
| Coupling | ✓ (event contract) |
| Observabilidad | ⚠ |

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ✗ |
| NFR-05 | ✓✓ |
| NFR-07 | ⚠ |
| NFR-09 | ✓ |

#### Complejidad operativa / Migración

Alta ops; excelente para Strangler **si** se acepta latencia en confirmación — **no** viable FTGO [PRD NFR-01].

---

### Opción 3: Híbrido REST + Kafka ✓

#### Descripción

| Canal | Uso |
|-------|-----|
| **REST/HTTPS/JSON** | Cliente ↔ Gateway ↔ Order/Delivery; UC-01, 02, 03, 05 |
| **Kafka** | Cross-BC: pago, dispatch, notificaciones |
| **REST/SDK** | Stripe, Maps, SendGrid [STK-06] |

Patrones: **Transactional Outbox**, at-least-once, consumidores idempotentes, saga coreografiada pago [Richardson Cap 3, Cap 5].

#### Evaluación dimensional

| Dimensión | Valoración |
|-----------|------------|
| Latencia | ✓✓ |
| Resiliencia | ✓✓ |
| Tolerancia fallos | ✓✓ |
| Consistency | ✓ Order / ⚠ entre BC |
| Escalabilidad | ✓✓ |
| Coupling | ✓ |
| Observabilidad | ✓ (HTTP + Kafka headers) |

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ✓ |
| NFR-02 | ✓ |
| NFR-03 | ✓ |
| NFR-04 | ✓ |
| NFR-05 | ✓✓ |
| NFR-06 | ✓ |
| NFR-07 | ✓ |
| NFR-08 | ✓ |
| NFR-09 | ✓✓ |
| NFR-10 | ✓ |

#### Complejidad operativa / Migración

Media-alta; Strangler con **mismos topics** monolito y MS [ADR-0001 F0–F2].

---

### Tabla comparativa

| Dimensión | REST | Kafka solo | Híbrido ✓ |
|-----------|------|------------|-----------|
| Latencia | ⚠ | ✗ | ✓✓ |
| Resiliencia | ⚠ | ✓✓ | ✓✓ |
| Tolerancia fallos | ✗ | ✓✓ | ✓✓ |
| Consistency | ⚠ | ⚠ | ✓ |
| Escalabilidad | ⚠ | ✓✓ | ✓✓ |
| Coupling | ✗ | ✓ | ✓ |
| Observabilidad | ✓ | ⚠ | ✓ |
| Strangler | ⚠ | ✓ | ✓✓ |

---

## 4. Decisión

**Opción 3 — Híbrido REST + Kafka.**

### Reglas normativas

1. **Prohibido** REST síncrono MS↔MS salvo excepción documentada en anexo (ninguna en v1).
2. **Writer único** por aggregate: solo Order Service (o monolito en ruta Strangler) publica `Order*` [evita duplicados Strangler].
3. **Billing** (monolito F1 o MS F2) publica únicos `Payment*` en `ftgo.payment.events`.
4. **correlation-id** en HTTP y headers Kafka [NFR-08].

### Topología Kafka (C4)

| Topic | Productores | Consumidores |
|-------|-------------|--------------|
| `ftgo.order.events` | Order MS / monolito (ruta activa) | Delivery, Notification, Kitchen |
| `ftgo.payment.events` | Billing monolito (F1), Billing MS (F2) | Order, Notification |
| `ftgo.delivery.events` | Delivery MS | Order, Notification |
| `ftgo.kitchen.commands` | Kitchen Service | Order Service |

**Key:** `orderId`. **Broker:** Apache Kafka — `ContainerQueue` en C4.

### Calendario alineado ADR-0001

| Fase | REST | Kafka |
|------|------|-------|
| F0 | Gateway → monolito | Cluster + outbox monolito piloto |
| F1 | Gateway → Order/Delivery | MS + monolito consumen/producen según ruta |
| F2 | + APIs Billing, etc. | Billing MS productor |
| F3 | Sin monolito en topics | Solo MS |

### Trade-offs

| Aceptamos | Por |
|-----------|-----|
| Ops Kafka + REST | NFR-05, NFR-02 |
| Eventualidad notificaciones | NFR-01 en commit local |
| Outbox obligatorio | NFR-07 |

### Coherencia ADR-0001

- F1 sin Billing MS **no bloquea** NFR-05: monolito + `payment.events`.
- C4 **debe** dibujar Kafka y Rel `"Kafka"` hacia Order, Billing, Delivery, Notification, Monolito adaptador.

---

## 5. Consecuencias

### Positivas

- NFR-01 + NFR-05 simultáneos [FSD UC-01, UC-04 GWT].
- Notification desacoplada (C-07).
- Strangler sin duplicar APIs MS↔monolito para cada cambio de estado.
- C4 coherente con decisión.

### Negativas (reales)

1. **Dos stacks** IPC — errores de modelo (publicar sin outbox).
2. **Duplicados** at-least-once — bug en NFR-07 si no hay idempotencia.
3. **Lag visible** notificaciones vs pantalla Order.
4. **Debug** cross-protocol más caro.
5. **Schema drift** rompe monolito y MS en paralelo.
6. **Costo Kafka** desde F0 aunque productores MS sean F1.
7. **Prohibición REST MS↔MS** requiere disciplina en code review.
8. **Incidentes consumer lag** en pico 5× — riesgo NFR-02 si no se escala consumidores.

---

## 6. Follow-ups

| ID | Acción | Fase |
|----|--------|------|
| FU-01 | Catálogo eventos v1 + AsyncAPI | F0 |
| FU-02 | Outbox Order + monolito/Billing | F0 piloto / F1 |
| FU-03 | Cluster Kafka + ACLs | F0 |
| FU-04 | correlation-id HTTP→Kafka | F0 |
| FU-05 | `processed_events` idempotencia | F1 |
| FU-06 | **Actualizar `c4_container.mmd`** | F0 |
| FU-07 | Caos Stripe [FSD UC-04] | F1 |
| FU-08 | ACL monolito ↔ ADR-0001 FU-04 | F1 |
| FU-09 | Checklist PR: no REST MS↔MS | F1 |

---

## Referencias

- Richardson Cap 3, Cap 5
- `docs/PRD.md`, `docs/FSD.md`, `docs/adr/0001-descomposicion-microservicios.md` v1.1

---

*ADR-0002 v1.1 Accepted — Kafka obligatorio en C4 Container.*
