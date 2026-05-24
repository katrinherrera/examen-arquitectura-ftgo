# FSD — FTGO Platform

| Campo | Valor |
|-------|-------|
| **Producto** | FTGO (Food To Go) |
| **Versión** | 1.1 |
| **Estado** | Alineado a PRD v1.1 |
| **Documento upstream** | `docs/PRD.md` |
| **Origen** | [Brief §A.4], [US-01]–[US-03], [PRD RF-01]–[PRD RF-03] |
| **Cambios v1.1** | GWT ampliados (rutas alternas); máquina de estados acotada; UC-05 alineado a FA-05.2; cierre `DELIVERED`; matriz NFR×UC |

---

## Introducción

Este **Functional Specification Document (FSD)** especifica el comportamiento funcional de FTGO derivado del **PRD v1.1**. Incluye **cinco casos de uso** obligatorios del laboratorio, trazados a user stories oficiales y flujos derivados del brief (pago resiliente, tracking).

**Alcance:**

- Transiciones del aggregate **Order** bajo [PRD NFR-07].
- Sin prescripción de broker ni productos (ADR-0001, ADR-0002) [PRD ASM-03].
- Convivencia **Strangler Fig** monolito / microservicios [PRD NFR-09] [PRD ASM-02].

**Regla de granularidad:** cancelación, reasignación courier y notificaciones son **flujos alternativos**, no UCs nuevos. Gestión de menús (C-02) queda fuera del mínimo de este FSD [PRD §5.1].

### Máquina de estados — Order (subset FSD)

Estados usados en los UCs (orden lineal salvo ramas):

| Estado | UC que lo establece |
|--------|---------------------|
| `DRAFT` | UC-01 (carrito) |
| `CONFIRMED` | UC-01 paso 4 |
| `TICKET_ISSUED` | UC-01 paso 6 |
| `ACCEPTED` / `REJECTED` | UC-02 |
| `PREPARING` / `READY` | UC-02 |
| `DELIVERY_ASSIGNED` | UC-03 |
| `IN_DELIVERY` | UC-03 (recogida) |
| `DELIVERED` | UC-03 paso 7 |
| `CANCELLED` | FA-01.4, FA-02.1 |

**Invariante [PRD NFR-07]:** no existe `DELIVERED` sin haber pasado por `READY` y `IN_DELIVERY`.

---

## Tabla resumen de casos de uso

| UC | Nombre | Actor primario | Capacidad PRD | Aggregate / BC | Origen | NFRs |
|----|--------|----------------|---------------|----------------|--------|------|
| UC-01 | Tomar pedido | Consumidor (STK-01) | C-03; C-01 (contexto) | Order / Order Taking | [US-01] [PRD RF-01] | NFR-01, 02, 03, 07, 08 |
| UC-02 | Aceptar o rechazar ticket cocina | Restaurante (STK-02) | C-04 | Order / Kitchen | [US-02] [PRD RF-02] | NFR-01, 03, 07 |
| UC-03 | Aceptar o rechazar asignación entrega | Courier (STK-03) | C-05 | Delivery + Order | [US-03] [PRD RF-03] | NFR-01, 06, 07 |
| UC-04 | Procesar pago del pedido | Consumidor (STK-01) * | C-06 | Payment / Billing | [US-01] [PRD NFR-05] | NFR-05, 08, 10 |
| UC-05 | Consultar tracking tiempo real | Consumidor (STK-01) | C-05; C-07 (lectura) | Proyección Delivery | [Brief §A.4] [PRD NFR-04] | NFR-04, 01 |

\* UC-04: el consumidor inicia checkout; el cobro lo ejecuta el sistema Billing (pasos automáticos).

### Trazabilidad PRD → FSD

| PRD | FSD |
|-----|-----|
| RF-01 | UC-01, UC-04 |
| RF-02 | UC-02 |
| RF-03 | UC-03 |
| NFR-04 | UC-05 |
| NFR-05 | UC-01 (FA-01.3), UC-04 |
| NFR-08 | UC-01, UC-04 (correlation-id) |

### Matriz NFR × UC

| NFR | UC-01 | UC-02 | UC-03 | UC-04 | UC-05 |
|-----|:-----:|:-----:|:-----:|:-----:|:-----:|
| NFR-01 | ✓ | ✓ | ✓ | | ✓ |
| NFR-02 | ✓ | | | | |
| NFR-03 | ✓ | ✓ | | | |
| NFR-04 | | | | | ✓ |
| NFR-05 | ✓ | | | ✓ | |
| NFR-06 | | | ✓ | | |
| NFR-07 | ✓ | ✓ | ✓ | | |
| NFR-08 | ✓ | | | ✓ | |
| NFR-10 | | | | ✓ | |

---

## UC-01: Tomar pedido

| Campo | Valor |
|-------|-------|
| **Actor primario** | Consumidor (STK-01) |
| **Capacidad PRD** | C-03 Order Taking; C-01 Consumer Management |
| **Origen** | [US-01] [PRD RF-01] [Brief §A.4] |
| **Bounded context** | Order Taking |
| **Aggregate** | Order |

#### Precondiciones

- Consumidor autenticado [PRD C-01].
- Restaurante con menú disponible e ítems en stock [PRD C-02].
- Carrito con ≥ 1 ítem y precio calculado.
- Plataforma FTGO disponible (monolito o Order Service) [PRD NFR-09].

#### Flujo principal

1. El consumidor revisa carrito y dirección de entrega.
2. El sistema valida ítems, stock y horario del restaurante.
3. El consumidor confirma checkout.
4. El sistema persiste **Order** en `CONFIRMED` con `correlation-id` [PRD NFR-08] [PRD NFR-07].
5. El sistema invoca **UC-04** (cobro).
6. Con pago `CAPTURED` o `PAYMENT_PENDING` [PRD NFR-05], Order → `TICKET_ISSUED`.
7. El sistema notifica restaurante (C-07) y responde al consumidor con `orderId` y estado [PRD NFR-01].

#### Flujos alternativos

**FA-01.1 — Ítem no disponible:** en paso 2, rechazo parcial; consumidor ajusta carrito; retorno a paso 1.

**FA-01.2 — Dirección fuera de cobertura:** en paso 1; no se crea Order confirmado; permanece `DRAFT` o no persiste.

**FA-01.3 — Pago pendiente (Stripe caído):** UC-04 FA-04.1; Order `CONFIRMED` + pago pendiente; emisión `TICKET_ISSUED` permitida [PRD NFR-05] [PRD NFR-03].

**FA-01.4 — Cancelación consumidor:** antes de `TICKET_ISSUED`; Order → `CANCELLED`; reverso vía UC-04 FA-04.3 si hubo captura.

**FA-01.5 — Strangler → monolito:** misma semántica de estados [PRD NFR-09].

#### Postcondiciones

- **Éxito:** `TICKET_ISSUED` (o `CONFIRMED` + pago pendiente en FA-01.3).
- **Fallo:** `DRAFT` / `CANCELLED`; sin ticket para UC-02.
- `correlation-id` en traza [PRD NFR-08].

#### Given/When/Then

**Escenario principal**

- **Given:** un consumidor autenticado con carrito válido en restaurante abierto
- **When:** confirma el pedido en checkout y el pago es autorizado
- **Then:** existe un Order en `TICKET_ISSUED` con cobro registrado y ticket visible para el restaurante

**Escenario pago pendiente [PRD NFR-05]**

- **Given:** un Order en `CONFIRMED` y Stripe no disponible
- **When:** el sistema ejecuta UC-04 y recibe timeout del PSP
- **Then:** el Order permanece válido, el pago queda pendiente y el pedido puede emitir ticket según FA-01.3

---

## UC-02: Aceptar o rechazar ticket de cocina

| Campo | Valor |
|-------|-------|
| **Actor primario** | Restaurante (STK-02) |
| **Capacidad PRD** | C-04 Order Fulfillment / Kitchen |
| **Origen** | [US-02] [PRD RF-02] |
| **Bounded context** | Kitchen |
| **Aggregate** | Order |

#### Precondiciones

- Order en `TICKET_ISSUED` del restaurante del actor.
- Terminal cocina autenticada.
- `restaurantId` del ticket = actor.

#### Flujo principal

1. El restaurante consulta cola de tickets.
2. Selecciona ticket y elige **Aceptar**.
3. Order: `TICKET_ISSUED` → `PREPARING` [PRD NFR-07].
4. Notificación al consumidor (C-07).
5. Al marcar **Listo**, Order → `READY`.
6. El sistema inicia dispatch para UC-03.

#### Flujos alternativos

**FA-02.1 — Rechazo:** en paso 2, **Rechazar** → `REJECTED` → `CANCELLED`; notificación consumidor; UC-04 FA-04.3 si aplica.

**FA-02.2 — Sin respuesta:** recordatorio a restaurante; escalación STK-04 [Brief §A.2]; estado sin cambio hasta aceptar/rechazar.

**FA-02.3 — Pago pendiente:** aceptación permitida según política FTGO alineada a FA-01.3 [PRD NFR-03].

#### Postcondiciones

- **Aceptación:** `PREPARING` o `READY`.
- **Rechazo:** `CANCELLED`.
- Invariantes Order [PRD NFR-07].

#### Given/When/Then

**Aceptar**

- **Given:** un Order en `TICKET_ISSUED` para el restaurante autenticado
- **When:** el restaurante acepta el ticket
- **Then:** el Order pasa a `PREPARING` y el consumidor recibe notificación

**Rechazar [US-02]**

- **Given:** un Order en `TICKET_ISSUED` para el restaurante autenticado
- **When:** el restaurante rechaza el ticket
- **Then:** el Order queda `CANCELLED` y el consumidor es notificado

---

## UC-03: Aceptar o rechazar asignación de entrega

| Campo | Valor |
|-------|-------|
| **Actor primario** | Courier (STK-03) |
| **Capacidad PRD** | C-05 Delivery |
| **Origen** | [US-03] [PRD RF-03] |
| **Bounded context** | Delivery |
| **Aggregate** | Delivery (1:1 con Order en entrega) |

#### Precondiciones

- Order en `READY`.
- Oferta de entrega para el courier.
- Courier autenticado y disponible.

#### Flujo principal

1. El courier recibe oferta (origen, destino, ETA) [PRD C-05].
2. Revisa detalle y elige **Aceptar**.
3. Sistema asigna Delivery; Order → `DELIVERY_ASSIGNED`.
4. El courier recoge el pedido en restaurante; Order → `IN_DELIVERY`.
5. Notificaciones consumidor y restaurante (C-07).
6. Posiciones GPS alimentan proyección de UC-05.
7. El courier confirma entrega en destino; Order → `DELIVERED` [derivado C-05 — completar entrega].

#### Flujos alternativos

**FA-03.1 — Rechazo:** oferta al pool; re-dispatch sin cancelar Order.

**FA-03.2 — Timeout oferta:** reasignación automática [Brief §A.4].

**FA-03.3 — Reasignación STK-04:** igual que FA-03.1 con prioridad operativa.

**FA-03.4 — Sin couriers:** Order en `READY`; alerta back office; notificación demora (C-07).

#### Postcondiciones

- **Éxito:** Order `DELIVERED`; Delivery cerrado.
- **Rechazo/reenvío:** Order ≥ `READY` hasta asignación exitosa.
- [PRD NFR-06] — componente Delivery escalable.

#### Given/When/Then

**Aceptar [US-03]**

- **Given:** un Order en `READY` y una oferta de entrega para el courier autenticado
- **When:** el courier acepta la entrega
- **Then:** el Order pasa a `DELIVERY_ASSIGNED` y puede transicionar a `IN_DELIVERY`

**Rechazar [US-03]**

- **Given:** una oferta de entrega pendiente para el courier
- **When:** el courier rechaza la entrega
- **Then:** la oferta vuelve al pool y el Order permanece `READY` para otra asignación

---

## UC-04: Procesar pago del pedido

| Campo | Valor |
|-------|-------|
| **Actor primario** | Consumidor (STK-01) — inicia; **Sistema FTGO / Billing** ejecuta |
| **Capacidad PRD** | C-06 Billing & Accounting |
| **Origen** | [US-01] [Brief §A.4] [PRD NFR-05] [PRD NFR-10] |
| **Bounded context** | Billing |
| **Aggregate** | Payment |

**Granularidad:** UC separado por BC Billing y objetivo de cobro [Regla laboratorio].

#### Precondiciones

- Order en `CONFIRMED` (UC-01 paso 4).
- Token de pago Stripe; sin PAN en FTGO [PRD NFR-10].
- Mismo `correlation-id` que Order [PRD NFR-08].

#### Flujo principal

1. Billing recibe `orderId`, monto, token (checkout UC-01).
2. Invoca Stripe (STK-06) con clave de idempotencia.
3. Stripe captura pago.
4. Payment → `CAPTURED`; notifica a Order.
5. UC-01 avanza a `TICKET_ISSUED` si aplica.

#### Flujos alternativos

**FA-04.1 — Stripe no disponible:** Payment `PAYMENT_PENDING`; Order no se revierte [PRD NFR-05]; reintento asíncrono.

**FA-04.2 — Tarjeta declinada:** Payment `FAILED`; Order `CONFIRMED` sin ticket hasta nuevo pago.

**FA-04.3 — Reembolso:** tras cancelación UC-01/UC-02; Payment `REFUNDED`.

**FA-04.4 — Billing en monolito:** comportamiento equivalente [PRD ASM-02].

#### Postcondiciones

- `CAPTURED` | `PAYMENT_PENDING` | `FAILED` | `REFUNDED`.
- PCI: sin PAN persistido [PRD NFR-10].

#### Given/When/Then

**Cobro exitoso**

- **Given:** un Order `CONFIRMED` con token de pago válido
- **When:** Billing procesa el cobro y Stripe responde éxito
- **Then:** Payment queda `CAPTURED` y Order recibe confirmación de pago

**Resiliencia Stripe [PRD NFR-05]**

- **Given:** un Order `CONFIRMED` listo para cobro
- **When:** Billing invoca Stripe y obtiene error 5xx o timeout
- **Then:** Payment queda `PAYMENT_PENDING`, el Order no se elimina y se programa reintento

---

## UC-05: Consultar tracking en tiempo real

| Campo | Valor |
|-------|-------|
| **Actor primario** | Consumidor (STK-01) |
| **Capacidad PRD** | C-05 Delivery; C-07 (opcional) |
| **Origen** | [Brief §A.4] [PRD NFR-04] |
| **Bounded context** | Delivery (lectura) |
| **Aggregate** | N/A — proyección de lectura |

**Granularidad:** objetivo de consulta distinto a UC-01 [Regla laboratorio].

#### Precondiciones

- Consumidor autenticado como titular del Order **o** token de seguimiento válido.
- Order existe y no está `CANCELLED`.

#### Flujo principal

1. El consumidor abre seguimiento del `orderId`.
2. El sistema lee estado Order y última posición Delivery.
3. Si Order ∈ {`DELIVERY_ASSIGNED`, `IN_DELIVERY`, `DELIVERED`}: consulta ETA/ruta vía Google Maps (STK-06) si disponible.
4. Respuesta: estado textual, mapa/posición, ETA.
5. Actualización periódica hasta `DELIVERED`.

#### Flujos alternativos

**FA-05.1 — Degradado [PRD NFR-04]:** última posición cacheada + aviso; no afecta UC-01/UC-03.

**FA-05.2 — Pre-entrega:** Order en `TICKET_ISSUED` … `READY`; muestra estado cocina sin mapa (enlace UC-02).

**FA-05.3 — Latencia lectura:** respuesta cacheada; reintento background [PRD NFR-01] en lectura.

#### Postcondiciones

- Consumidor informado; SLA tracking 99.5% con degradación [PRD NFR-04].
- Sin escritura en aggregate Order.

#### Given/When/Then

**Tracking en entrega**

- **Given:** un Order en `IN_DELIVERY` con posición GPS reciente del courier
- **When:** el consumidor abre la pantalla de tracking
- **Then:** ve estado, ubicación (o cacheada) y ETA cuando Maps está disponible

**Modo degradado [PRD NFR-04]**

- **Given:** un Order en `IN_DELIVERY` y Google Maps no disponible
- **When:** el consumidor consulta tracking
- **Then:** ve la última posición conocida y un mensaje de servicio degradado

---

## Diagrama de dependencias

```
UC-01 ──► UC-04
  │
  └──► UC-02 ──► UC-03 ──► UC-05
         (FA-05.2 enlaza estados pre-entrega)
```

---

## Criterios de aceptación del FSD (v1.1)

- [x] ≥ 5 UCs con estructura completa
- [x] GWT en todos los UCs (+ escenarios alternos clave)
- [x] Trazabilidad PRD / US / Brief / NFR
- [x] Sin UCs inventados (menús, loyalty excluidos)
- [x] Coherencia estados Order y PRD NFR-07
- [x] UC-05 precondiciones compatibles con FA-05.2

---

## Referencias downstream

| FSD | Artefacto |
|-----|-----------|
| UC-01, UC-02, UC-04 | `docs/adr/0002-ipc-event-driven.md` |
| Todos | `docs/adr/0001-descomposicion-microservicios.md` |
| Todos | `docs/diagrams/c4_*.mmd` |

---

*FSD v1.1 — Sincronizado con `docs/PRD.md` tabla NFR→UC.*
