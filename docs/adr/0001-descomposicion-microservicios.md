# ADR-0001: Estrategia de descomposición del monolito FTGO

| Campo | Valor |
|-------|-------|
| **Status** | Accepted |
| **Versión** | 1.1 |
| **Fecha** | 2026-05-24 |
| **Decisores** | Equipo de arquitectura (STK-05) |
| **Trazabilidad** | [PRD NFR-09] [PRD NFR-06] [Brief §A.4] [Richardson Cap 2] [Richardson Cap 3] |
| **Relacionado** | `docs/PRD.md` v1.1, `docs/FSD.md` v1.1, **ADR-0002** [Accepted] |
| **Cambios v1.1** | Matriz coherencia ADR-0002; F1 Billing+Kafka; NFR-05 desde F0; coste Kafka explícito |

---

## 1. Título y status

**ADR-0001 — Estrategia de descomposición del monolito FTGO mediante Strangler Fig e implementación híbrida incremental por capability de negocio.**

- **Status:** Accepted
- **Supersede:** N/A

---

## 2. Contexto

FTGO es un marketplace de delivery (monolito Java WAR) con **siete capacidades** C-01…C-07 [PRD §3]. Migración **18–24 meses**, **sin big-bang** [PRD NFR-09] [Brief §A.4], vía **Strangler Fig** [Richardson Cap 3].

**Drivers:**

| Driver | NFR / origen |
|--------|----------------|
| Pico 5× almuerzo/cena | NFR-02, NFR-06 |
| Camino crítico pedidos | NFR-03 |
| Desacoplar billing / Stripe | NFR-05, [US-01] |
| Aggregate Order consistente | NFR-07 |
| Observabilidad distribuida | NFR-08 |
| Rollback por oleada | NFR-09 |

**Restricciones:** Java/Spring Boot [PRD RES-01]; UCs UC-01…UC-05 [FSD]; IPC definida en **ADR-0002** (REST + Kafka).

**Problema:** ¿Cómo descomponer el monolito maximizando NFRs sin big-bang ni riesgo operativo descontrolado?

---

## 3. Opciones consideradas

Strangler Fig es **común a las tres** opciones [PRD NFR-09]. La decisión es el **ritmo y la forma** de descomposición.

### Resumen comparativo

| Dimensión | Op.1 MS paralelo | Op.2 Modular monolith | Op.3 Híbrido incremental ✓ |
|-----------|------------------|----------------------|------------------------------|
| Escalabilidad NFR-06 | ✓✓ | ✗ | ✓✓ (por fases) |
| Complejidad operativa | Alta (temprana) | Baja | Media → alta |
| Latencia NFR-01 | ⚠ | ✓ | ✓ (gateway) |
| Resiliencia NFR-03/05 | ✓ | ✗ | ✓ |
| Coupling | Bajo entre MS | Bajo código / alto deploy | Bajo progresivo |
| Costo operativo | Alto | Bajo | Medio-alto (+ Kafka ADR-0002) |
| Migración NFR-09 | ⚠ por servicio | ✗ objetivo final | ✓✓ |

---

### Opción 1: Microservicios por capability (extracción agresiva en paralelo)

#### Descripción

Extraer **los seis servicios** en ventanas cortas superpuestas (Order, Delivery, Billing, Notification, Consumer, Restaurant), cada uno con DB propia, detrás de Strangler [Richardson Cap 2]. Diferencia con Op.3: **poca convivencia** monolito–MS por dominio; presión por “terminar” extracciones.

#### Pros

- NFR-06 rápido en toda la plataforma.
- Aislamiento PCI en Billing temprano (NFR-10).
- Equipos por capability.

#### Contras

- Complejidad operativa **máxima** en meses 4–12 (6 pipelines, 6 DBs, Kafka ADR-0002).
- Riesgo **big-bang por dominio** y rollback difícil [NFR-09].
- NFR-07 frágil sin outbox maduro (depende ADR-0002 FU-02).
- “Microservice envy” [Richardson Cap 2].

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ⚠ |
| NFR-02 | ✓ |
| NFR-03 | ⚠ |
| NFR-05 | ✓ (si Billing temprano) |
| NFR-06 | ✓✓ |
| NFR-07 | ⚠ |
| NFR-08 | ⚠ (madurez) |
| NFR-09 | ⚠ |
| NFR-10 | ✓ |

#### Complejidad operativa

**Alta** desde mes 4.

#### Impacto en migración

Strangler formal, pero **alta probabilidad de corte coordinado** multi-equipo; poco realista para FTGO 18–24 meses sin incidentes mayores.

---

### Opción 2: Modular monolith temporal

#### Descripción

Módulos Maven/Gradle por capability **sin** procesos separados en 18–24 meses. Strangler no expone microservicios nuevos.

#### Pros

- NFR-01, NFR-07 simples.
- Ops mínima.

#### Contras

- **Incumple NFR-06** y objetivo PRD de microservicios.
- NFR-05 acoplado en un WAR.
- Doble migración futura.
- NFR-09 no alcanza arquitectura objetivo.

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ✓ |
| NFR-02 | ✗ |
| NFR-05 | ✗ |
| NFR-06 | ✗ |
| NFR-07 | ✓ |
| NFR-09 | ✗ |

#### Complejidad operativa

**Baja**.

#### Impacto en migración

Válido solo como **táctica interna** dentro del monolito previa a Op.3; **no** decisión final.

---

### Opción 3: Híbrido incremental (Strangler + MS por capability en oleadas) ✓

#### Descripción

Strangler + extracción **secuencial por valor y NFR**:

| Fase | Despliegue | Monolito | IPC [ADR-0002] |
|------|------------|----------|----------------|
| **F0** | Gateway, tracing, **Kafka infra** | 100% tráfico app | REST gateway→monolito; outbox monolito→Kafka (piloto) |
| **F1** | Order + Delivery MS | Billing, C-01/02/07 | REST UX; eventos dominio; **billing lógica en monolito** publica `payment.events` [PRD ASM-02] |
| **F2** | Billing, Notification, Consumer, Restaurant | Rutas residuales | Billing MS productor PSP |
| **F3** | Apagado monolito | 0% | Solo MS |

Módulos internos monolito (Op.2) permitidos en F0–F1 como puente de código.

#### Pros

- NFR-09 con rollback por ruta.
- NFR-06 en F1 (Order, Delivery) para pico.
- NFR-05 desde F0 vía eventos aunque Billing sea monolito [PRD ASM-02] + ADR-0002.
- Alineado FSD (UC-01 F1, UC-04 monolito→evento, UC-05 F1).

#### Contras

- Arquitectura dual 12–18 meses.
- Ops media-alta + **Kafka** (coste ADR-0002).
- Riesgo datos divergentes monolito ↔ Order DB.
- F2 con 4 extracciones — cuello de botella de entrega.

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 | ✓ |
| NFR-02 | ✓ (F1) |
| NFR-03 | ✓ |
| NFR-04 | ✓ (Delivery F1) |
| NFR-05 | ✓ (eventos F0+; Billing MS F2) |
| NFR-06 | ✓✓ |
| NFR-07 | ✓ (Order dueño F1) |
| NFR-08 | ✓ (F0) |
| NFR-09 | ✓✓ |
| NFR-10 | ✓ (F2 Billing) |

#### Complejidad operativa

Media F0–F1; **alta** F2–F3.

#### Impacto en migración

**Recomendada** para FTGO: valida Strangler y NFRs antes de multiplicar servicios [Richardson Cap 3 — evolve].

---

## 4. Decisión

**Opción 3 — Híbrido incremental** con Strangler, oleadas F0–F3, e **IPC híbrida REST+Kafka** según ADR-0002 [Accepted].

### Trade-offs aceptados

| Aceptamos | Rechazamos | Por qué |
|-----------|------------|---------|
| Deuda dual monolito+MS | Big-bang por dominio | NFR-09 |
| Kafka desde F0 (infra) | REST-only inter-BC | NFR-05, ADR-0002 |
| Billing físico en F2 | Extraer Consumer antes que Order | NFR-02, UC-01 |
| 6 servicios al final | Modular monolith final | NFR-06, PRD |

### Mapa capability → servicio

| Capability | Servicio | Fase |
|------------|----------|------|
| C-03, C-04 | Order Service | F1 |
| C-05 | Delivery Service | F1 |
| C-06 | Billing Service | F2 |
| C-07 | Notification Service | F2 |
| C-01 | Consumer Service | F2 |
| C-02 | Restaurant Service | F2 |

### Coherencia con ADR-0002

| ADR-0001 | ADR-0002 |
|----------|----------|
| F0 gateway | REST cliente→monolito |
| F0 Kafka cluster | Outbox monolito, sin consumidores MS aún |
| F1 Order/Delivery MS | REST write-path UX; Kafka cross-BC |
| F1 Billing en monolito | Publica `ftgo.payment.events` |
| F2 Billing MS | Productor PSP + mismos topics |
| C4 Container | `ContainerQueue` Kafka obligatorio |

---

## 5. Consecuencias

### Positivas

- NFR-09 cumplible con flags y rollback.
- Valor F1 en pedido/entrega/tracking [FSD UC-01, UC-03, UC-05].
- Richardson Cap 2 + Cap 3 alineados.
- Base C4: Monolito Legacy + servicios por fase.

### Negativas (reales)

1. **Deuda dual** monolito + MS + topics Kafka — tres superficies de fallo.
2. **Costo ops** superior (gateway, 2→6 servicios, broker 24×7).
3. **NFR-01 inconsistente** entre rutas hasta F3.
4. **E2E** matriz monolito × MS × flags × eventos.
5. **Divergencia de datos** monolito vs Order DB (NFR-07).
6. **F2 sobrecargado** — riesgo de retraso Billing/Notification.
7. **Dependencia de ADR-0002**: sin outbox, la descomposición F1 falla en producción.
8. **Equipo pequeño**: riesgo de subestimar on-call Kafka + K8s.

---

## 6. Follow-ups

| ID | Acción | Plazo | Artefacto |
|----|--------|-------|-----------|
| FU-01 | Matriz rutas Strangler + rollback | F0 | C4 / runbook |
| FU-02 | Ejecutar criterios ADR-0002 (outbox F0 piloto, Kafka F0) | F0 | `0002-ipc-event-driven.md` |
| FU-03 | correlation-id en gateway [NFR-08] | F0 | Estándar |
| FU-04 | Order Service + ACL monolito | F1 | C4 |
| FU-05 | Delivery + prueba carga 5× [NFR-02] | F1 | Informe |
| FU-06 | SLA máximo deuda dual por dominio | F1 | Roadmap |
| FU-07 | Billing MS + PCI [NFR-10] | F2 | Checklist |
| FU-08 | Apagado monolito [NFR-09] | F3 | Acta |

---

## Referencias

- Richardson, *Microservices Patterns*, Cap 2, Cap 3
- `docs/PRD.md`, `docs/FSD.md`, `docs/adr/0002-ipc-event-driven.md`
- [Brief §A.4]

---

*ADR-0001 v1.1 Accepted — Par con ADR-0002 para C4 Container.*
