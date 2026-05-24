# ADR-0001: Estrategia de descomposición del monolito FTGO

| Campo | Valor |
|-------|-------|
| **Status** | Accepted |
| **Fecha** | 2026-05-24 |
| **Decisores** | Equipo de arquitectura (STK-05) |
| **Trazabilidad** | [PRD NFR-09] [PRD NFR-06] [Brief §A.4] [Richardson Cap 2] [Richardson Cap 3] |
| **Relacionado** | `docs/PRD.md` v1.1, `docs/FSD.md` v1.1, ADR-0002 (comunicación) |

---

## 1. Título y status

**ADR-0001 — Estrategia de descomposición del monolito FTGO mediante Strangler Fig e implementación híbrida incremental por capability de negocio.**

- **Status:** Accepted
- **Supersede:** N/A (primera decisión de descomposición)

---

## 2. Contexto

FTGO es un marketplace de delivery (monolito Java WAR) con siete **capacidades oficiales** (C-01…C-07) [PRD §3]. El negocio exige migración **incremental en 18–24 meses** sin big-bang [PRD NFR-09] [Brief §A.4], usando el patrón **Strangler Fig**: una fachada enruta tráfico al monolito legacy o a servicios nuevos [Richardson Cap 3].

**Drivers de la decisión:**

| Driver | NFR / origen |
|--------|----------------|
| Escalar pedido y entrega en horas pico (5×) | NFR-02, NFR-06 [Brief §A.4] |
| Disponibilidad camino crítico de pedidos | NFR-03 |
| Reducir acoplamiento billing ↔ order | NFR-05, [US-01] |
| Mantener consistencia del aggregate Order | NFR-07 |
| Observabilidad en arquitectura distribuida | NFR-08 |
| Cumplir ventana de migración con rollback | NFR-09 |

**Restricciones:**

- No inventar capacidades fuera de C-01…C-07 [PRD §3].
- Core en Java/Spring Boot [PRD RES-01].
- Fases PRD: F0 fachada; F1 Order + Delivery; F2 Billing, Notification, Consumer, Restaurant [PRD §5.3].
- UCs críticos: UC-01 (pedido), UC-03 (courier), UC-04 (pago) [FSD].

**Problema a resolver:** ¿Cómo descomponer el monolito para maximizar NFRs sin violar migración incremental ni multiplicar riesgo operativo?

---

## 3. Opciones consideradas

Se evalúan tres estrategias reales de descomposición. En todas se asume **Strangler Fig** como mecanismo de migración (no es opcional — [PRD NFR-09]).

### Dimensión de comparación (resumen)

| Dimensión | Opción 1 — MS por capability | Opción 2 — Modular monolith | Opción 3 — Híbrido incremental ✓ |
|-----------|------------------------------|-----------------------------|----------------------------------|
| Escalabilidad (NFR-06) | ✓✓ | ✗ | ✓✓ (fases) |
| Complejidad operativa | Alta | Baja | Media creciente |
| Latencia (NFR-01) | Riesgo red | ✓ | Controlada por fachada |
| Resiliencia (NFR-03/05) | ✓ | ✗ acoplamiento | ✓ |
| Coupling | Bajo entre BC | Bajo interno, alto despliegue único | Bajo progresivo |
| Costo operativo | Alto | Bajo | Medio |
| Migración (NFR-09) | Media (big-bang por servicio) | No alcanza objetivo MS | ✓✓ |

---

### Opción 1: Microservicios por capability de negocio

#### Descripción

Extraer **un microservicio por cada capacidad oficial** (Consumer, Restaurant, Order, Delivery, Billing, Notification) en oleadas, cada uno con base de datos propia, detrás de API Gateway Strangler. Bounded context = capability [Richardson Cap 2 — decompose by business capability]. El monolito se apaga por rutas hasta desaparecer.

Alineación FTGO: Order Service (C-03/C-04), Delivery Service (C-05), Billing Service (C-06), etc.

#### Pros

- Escalado independiente por servicio (NFR-06) [Richardson Cap 2].
- Aislamiento de fallos y despliegues por contexto.
- Equipos pueden alinearse a capabilities ( Conway-friendly ).
- Cumple visión objetivo del PRD (microservicios Java/Spring).

#### Contras

- **Alta complejidad operativa** desde oleadas tempranas: K8s, múltiples DBs, secretos, pipelines × 6.
- Riesgo de **“microservicios prematuros”** si se extrae Notification antes que Order [Richardson Cap 2 — anti-pattern].
- Latencia adicional en cadenas sync multi-hop (NFR-01).
- Datos distribuidos: riesgo a NFR-07 si no hay diseño de sagas (depende ADR-0002).
- Costo de infra y observabilidad multiplicado.

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 Latencia | ⚠ — más saltos de red |
| NFR-02 Carga | ✓ — escala Order/Delivery |
| NFR-03 Disponibilidad | ✓ — si circuit breakers |
| NFR-04 Tracking | ✓ — Delivery aislado |
| NFR-05 Stripe | ✓ — Billing aislado |
| NFR-06 Escalabilidad | ✓✓ |
| NFR-07 Consistencia | ⚠ — Order distribuido |
| NFR-08 Trazabilidad | ✓ — obligatorio |
| NFR-09 Migración | ⚠ — riesgo big-bang *por servicio* si oleadas mal planificadas |
| NFR-10 Compliance | ✓ — PCI en Billing |

#### Complejidad operativa

**Alta** desde F1: 2+ servicios + gateway + tracing + CI/CD paralelos.

#### Impacto en migración

Strangler aplicable, pero la tentación es **extraer demasiados servicios en paralelo** y perder rollback simple. Ventana 18–24 meses viable solo con disciplina estricta de oleadas.

---

### Opción 2: Modular monolith temporal (sin extracción a microservicios en el horizonte)

#### Descripción

Reestructurar el WAR monolito en **módulos Gradle/Maven** por capability (paquetes, límites de API interna, base de datos lógica separada por schema), **sin** procesos desplegables separados durante 18–24 meses. Strangler solo como refactor interno o no aplicado hacia servicios externos.

#### Pros

- **Menor complejidad operativa**: un despliegue, una DB física (o schemas), herramientas actuales.
- Mejor latencia local (NFR-01) — llamadas in-process.
- Refactor incremental de código con menor riesgo de red.
- Consistencia Order más simple (NFR-07) en una transacción.
- Costo operativo bajo.

#### Contras

- **No cumple NFR-06** (escalado independiente por componente) [PRD NFR-06].
- Sigue un **punto único de fallo** de despliegue; builds lentos persisten [Brief §A.1].
- No aísla Billing/Stripe del resto (NFR-05 más difícil).
- Al final del horizonte habría que **re-hacer extracción física** — doble trabajo.
- Strangler hacia “nada nuevo” no reduce lock-in tecnológico [Brief §A.1].
- Equipos siguen pisándose en un repo.

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 Latencia | ✓ |
| NFR-02 Carga | ✗ — escala todo el monolito |
| NFR-03 Disponibilidad | ⚠ — blast radius |
| NFR-04 Tracking | ⚠ |
| NFR-05 Stripe | ✗ — acoplamiento |
| NFR-06 Escalabilidad | ✗ |
| NFR-07 Consistencia | ✓ |
| NFR-08 Trazabilidad | ✓ — más simple |
| NFR-09 Migración | ✗ — no alcanza arquitectura objetivo |
| NFR-10 Compliance | ⚠ — scope PCI más amplio |

#### Complejidad operativa

**Baja** — favorable solo a corto plazo.

#### Impacto en migración

**No satisface** el objetivo de migración del laboratorio/PRD hacia microservicios. Puede ser etapa **subordinada** (módulos dentro del monolito), no estrategia final.

---

### Opción 3: Híbrido incremental (Strangler + microservicios por capability en oleadas)

#### Descripción

Combinar **Strangler Fig** [Richardson Cap 3] con extracción **progresiva por capability**, priorizando camino crítico y NFRs:

| Fase | Extracción | Monolito |
|------|------------|----------|
| F0 | API Gateway, flags, tracing (NFR-08) | 100% tráfico no piloto |
| F1 | **Order Service** (C-03, C-04 estado), **Delivery Service** (C-05) | Billing, Consumer, Restaurant, Notification |
| F2 | **Billing**, **Notification**, **Consumer**, **Restaurant** | Rutas residuales |
| F3 | Apagado monolito | 0% tráfico producción |

Opcional en F0–F1: **módulos internos** en monolito (preparación Opción 2) sin despliegue separado — puente de código, no estrategia final.

Cada ruta migrada: feature flag + rollback de enrutamiento [PRD NFR-09]. Comunicación entre servicios según ADR-0002.

#### Pros

- Cumple **NFR-09** con rollback por ruta [Richardson Cap 3].
- Alcanza **NFR-06** en F1 para Order/Delivery (pico 5×).
- Reduce riesgo vs “big-bang microservicios”: validación por oleada.
- Alineado a **FSD** (UC-01/03/04 en F1–F2) y fases PRD.
- Permite convivencia: billing en monolito con NFR-05 lógico [PRD ASM-02].
- Modularización interna opcional reduce deuda antes de cortar.

#### Contras

- **Período prolongado de arquitectura dual** (monolito + MS): mayor carga cognitiva y bugs de enrutamiento.
- Complejidad operativa **media → alta** a medida que crecen servicios.
- Riesgo de **inconsistencia de datos** entre monolito y Order DB si no hay anti-corruption layer [Richardson Cap 3].
- Latencia variable durante F1 (algunas rutas más lentas).
- Costo de gateway, doble mantenimiento temporal de código.

#### Impacto en NFRs

| NFR | Impacto |
|-----|---------|
| NFR-01 Latencia | ✓ — fachada optimizada; ⚠ en rutas híbridas |
| NFR-02 Carga | ✓ — F1 Order+Delivery |
| NFR-03 Disponibilidad | ✓ — rollback |
| NFR-04 Tracking | ✓ — Delivery F1 |
| NFR-05 Stripe | ✓ — Billing F2; lógica resiliente antes |
| NFR-06 Escalabilidad | ✓✓ — progresivo |
| NFR-07 Consistencia | ✓ — Order Service dueño aggregate |
| NFR-08 Trazabilidad | ✓ — F0 obligatorio |
| NFR-09 Migración | ✓✓ |
| NFR-10 Compliance | ✓ — Billing F2 |

#### Complejidad operativa

**Media** en F0–F1 (2 servicios + gateway); **alta** en F2 (6 servicios). Plan de SRE y observabilidad desde F0.

#### Impacto en migración

**Óptimo** para ventana 18–24 meses: valor de negocio temprano (pedido + entrega) sin extraer Consumer/Restaurant antes de tiempo [Richardson Cap 2 — evolve].

---

## 4. Decisión

**Se adopta la Opción 3: Híbrido incremental** — Strangler Fig + extracción de microservicios **por capability de negocio** en oleadas alineadas al PRD (F0→F3), usando módulos internos en monolito solo como **táctica preparatoria**, no como estado final.

### Justificación de trade-offs

| Trade-off | Elección | Por qué |
|-----------|----------|---------|
| Velocidad operativa vs escalabilidad | Escalabilidad progresiva | NFR-02 y NFR-06 son drivers del caso [Brief §A.4]; modular monolith solo no cierra el PRD |
| Simplicidad vs time-to-value | Oleadas F1 Order+Delivery | Máximo impacto en UC-01, UC-03, UC-05 y horas pico [FSD] |
| Consistencia vs autonomía | Order Service dueño del aggregate | NFR-07; integraciones vía ADR-0002 |
| Costo operativo vs riesgo big-bang | Híbrido con rollback | NFR-09 [Richardson Cap 3] |

**Por qué no Opción 1 pura:** extraer las 6 capabilities en paralelo eleva complejidad operativa antes de validar Strangler y NFRs en producción — anti-patrón “microservice envy” [Richardson Cap 2].

**Por qué no Opción 2 pura:** incumple NFR-06 y objetivos de migración del PRD; deja doble migración futura.

### Mapa capability → servicio (objetivo)

| Capability | Servicio | Fase extracción |
|------------|----------|-----------------|
| C-03, C-04 (estado Order) | Order Service | F1 |
| C-05 | Delivery Service | F1 |
| C-06 | Billing Service | F2 |
| C-07 | Notification Service | F2 |
| C-01 | Consumer Service | F2 |
| C-02 | Restaurant Service | F2 |

### Componentes Strangler (F0)

- API Gateway / reverse proxy con enrutamiento por path y feature flags.
- Monolito Legacy como System downstream hasta F3.
- Correlation ID obligatorio en gateway [PRD NFR-08].

---

## 5. Consecuencias

### Positivas

- Cumplimiento de **NFR-09** con migración reversible por ruta.
- **NFR-06** satisfecho para componentes críticos desde F1.
- Alineación con **Richardson** (decompose by capability + Strangler) [Cap 2, Cap 3].
- Trazabilidad clara PRD → FSD → servicios para C4.
- Equipos pueden entregar valor en mes 4–9 (F1) sin esperar F3.

### Negativas (reales)

1. **Deuda de arquitectura dual** durante 12–18 meses: dos modelos de datos y código para las mismas rutas; riesgo de regresiones en enrutamiento.
2. **Costo operativo superior** al monolito: múltiples pipelines, alertas, on-call por servicio; presión en STK-05 sin madurez DevOps.
3. **Latencia inconsistente** entre rutas monolito vs microservicio hasta completar F3 (NFR-01 en riesgo si gateway mal dimensionado).
4. **Complejidad de pruebas E2E** multiplicada: matriz monolito × MS × flags.
5. **Riesgo de datos divergentes** entre monolito y Order Service si la sincronización Strangler falla (impacto NFR-07).
6. **Curva de aprendizaje** del equipo en distributed systems antes de que todo el valor esté extraído.
7. **F2 concentrado**: cuatro servicios en una fase — posible cuello de botella de entrega si no se planifica capacidad.

### Riesgos mitigados (acciones en follow-ups)

- Anti-Corruption Layer en gateway hacia monolito.
- Contract tests entre F1 y monolito.
- Runbooks de rollback de flags [NFR-09].

---

## 6. Follow-ups

| ID | Acción | Responsable | Plazo | Artefacto |
|----|--------|-------------|-------|-----------|
| FU-01 | Definir matriz de rutas Strangler (path → monolito \| MS) y criterios de rollback | Arquitectura | F0 | ADR anexo / C4 |
| FU-02 | Aprobar ADR-0002 (sync vs eventos, NFR-05/07) antes de F1 | Arquitectura | F0 | `0002-event-driven-order-communication.md` |
| FU-03 | Implementar correlation ID + tracing en gateway (NFR-08) | Plataforma | F0 | Observabilidad |
| FU-04 | Extraer Order Service con DB propia y anti-corruption hacia monolito | Order team | F1 | C4 Container |
| FU-05 | Extraer Delivery Service; validar NFR-02 con prueba de carga 5× | Delivery team | F1 | Informe perf |
| FU-06 | Definir SLA de deuda dual (fecha máxima rutas híbridas por dominio) | Arquitectura | F1 | Roadmap |
| FU-07 | Extraer Billing; acotar PCI (NFR-10) | Billing team | F2 | ADR/compliance checklist |
| FU-08 | Apagar última ruta monolito; revisión post-mortem NFR-09 | Arquitectura | F3 | Acta cierre |

---

## Referencias

- Chris Richardson, *Microservices Patterns*, Cap 2 — Decomposition strategies
- Chris Richardson, *Microservices Patterns*, Cap 3 — Interprocess communication & API Gateway / Strangler
- `docs/PRD.md` — NFR-01…NFR-10, fases F0–F3
- `docs/FSD.md` — UC-01…UC-05
- [Brief §A.4] — Strangler, escalabilidad, migración 18–24 meses

---

*ADR-0001 Accepted — Base para diagramas C4 (Monolito Legacy + servicios por fase).*
