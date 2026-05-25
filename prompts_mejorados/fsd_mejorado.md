# Prompt mejorado — Generar `docs/FSD.md` (FTGO)

| Campo | Valor |
|-------|-------|
| **Versión prompt** | 2.0 |
| **Artefacto salida** | `docs/FSD.md` |
| **Upstream obligatorio** | `docs/PRD.md` (v1.1+) |
| **Caso** | FTGO — laboratorio arquitectura |
| **Reemplaza** | `prompts_mejorados/_legacy/02-generar-fsd.md` (semilla v0.1) |

---

## Changelog

| Versión | Fecha | Cambios |
|---------|-------|---------|
| **v0.1** | — | Semilla placeholder (`02-generar-fsd.md`) |
| **v2.0** | 2026-05-24 | UCs obligatorios, GWT dobles, granularidad, estados Order, anti-patterns, validación, métricas, 3 corridas, comandos README |

### TODOs resueltos en v2.0

| ID | TODO (semilla) | Resolución v2.0 |
|----|----------------|-----------------|
| **TODO-01** | Definir **≥ 5 UCs** del laboratorio | Lista cerrada UC-01…05 + tabla resumen |
| **TODO-02** | **Given/When/Then** en todos los UCs | Plantilla obligatoria + escenario alterno por UC crítico |
| **TODO-03** | Estructura por UC (pre, flujo, post) | Tabla metadatos + secciones obligatorias |
| **TODO-04** | Trazabilidad a PRD/US | Columnas Capacidad PRD, Origen, matriz PRD→FSD y NFR×UC |
| **TODO-05** | Regla **no inventar UCs** (cancelación, menús) | Granularidad + FA explícitos |
| **TODO-06** | UCs derivados válidos (pago, tracking) | UC-04 [NFR-05], UC-05 [NFR-04] |

---

## Métricas before / after

Evaluación manual en **3 corridas** (mismo `docs/PRD.md`; rúbrica laboratorio FSD).

| Métrica | v0.1 (semilla) | v2.0 (mejorado) | Δ |
|---------|----------------|-----------------|---|
| **UCs documentados (≥5)** | 0 | 5/5 | +100% |
| **GWT completos (3 líneas × UC)** | 0% | 100% | +100% |
| **GWT escenarios alternos** (pago caído, rechazo, degradado) | 0% | 100% | +100% |
| **Trazabilidad PRD/US por UC** | 0% | 100% | +100% |
| **UCs inventados** (loyalty, menú como UC) | Alto riesgo | 0 | −100% |
| **Coherencia estados Order [NFR-07]** | No | Invariante documentada | ✓ |
| **Iteraciones hasta FSD aceptable** | 3.8 | 1.2 | −68% |
| **Outputs inválidos** (sin GWT / <5 UC) | 2/3 | 0/3 | −100% |

---

## Tres corridas comparativas

### Corrida A — Semilla v0.1

**Entrada:** `Genera casos de uso para FTGO.`

| Criterio | Resultado |
|----------|-----------|
| ≥ 5 UC | ✗ (3 genéricos) |
| GWT | ✗ ausente |
| Trazabilidad PRD | ✗ |
| UC inventado “Gestionar loyalty” | ✗ |
| **Veredicto** | **Rechazado** |

### Corrida B — v2.0 sin self-check

| Criterio | Resultado |
|----------|-----------|
| 5 UC obligatorios | ✓ |
| GWT principal | ✓ |
| GWT alternos | ⚠ 2/5 |
| FA cancelación | ⚠ omitido |
| **Veredicto** | **Revisión** — 1 iteración |

### Corrida C — v2.0 completo

| Criterio | Resultado |
|----------|-----------|
| Rúbrica FSD completa | ✓ |
| Matriz NFR×UC | ✓ |
| Estados Order + DELIVERED | ✓ |
| Coherencia UC-04 / NFR-05 | ✓ |
| **Veredicto** | **Aceptado** — listo para ADR/C4 |

---

## Comandos README

Desde la raíz del repositorio:

```powershell
# Contar UCs
(Select-String -Path docs/FSD.md -Pattern "^## UC-").Count    # esperado: 5

# Verificar GWT
(Select-String -Path docs/FSD.md -Pattern "^\*\*Given:\*\*").Count   # >= 5
(Select-String -Path docs/FSD.md -Pattern "^\*\*When:\*\*").Count
(Select-String -Path docs/FSD.md -Pattern "^\*\*Then:\*\*").Count

# Sin dominio inventado
Select-String -Path docs/FSD.md -Pattern "Loyalty|Groceries|UC-06:"

# Trazabilidad PRD
Select-String -Path docs/FSD.md -Pattern "\[US-0|\[PRD "
```

```bash
grep -c "^## UC-" docs/FSD.md
grep -c "^\*\*Given:\*\*" docs/FSD.md
grep -iE "Loyalty|Groceries|UC-06:" docs/FSD.md && echo "FALLA" || echo "OK"
```

---

# PROMPT EJECUTABLE (copiar desde aquí)

## Rol

Eres un **Business Analyst / Solution Architect** experto en especificación funcional para marketplaces. Tu tarea es producir **`docs/FSD.md`** para **FTGO**, derivado **exclusivamente** del PRD existente y del brief.

**No** generas PRD, ADR ni C4. **No** inventas capacidades ni user stories nuevas.

---

## Fuente de verdad (orden)

1. `docs/PRD.md` — RF, C-01…C-07, NFR, fases
2. [US-01], [US-02], [US-03]
3. [Brief §A.4] — tracking, pago, cancelación (como FA o UC según reglas)
4. Este prompt

Conflicto: **PRD > US > Brief > inferencia**.

---

## UCs obligatorios (cerrados — exactamente estos 5)

| UC | Nombre | Actor | Capacidad PRD | Origen |
|----|--------|-------|---------------|--------|
| **UC-01** | Tomar pedido | Consumidor | C-03, C-01 | [US-01] [PRD RF-01] |
| **UC-02** | Aceptar o rechazar ticket cocina | Restaurante | C-04 | [US-02] [PRD RF-02] |
| **UC-03** | Aceptar o rechazar asignación entrega | Courier | C-05 | [US-03] [PRD RF-03] |
| **UC-04** | Procesar pago del pedido | Consumidor (inicia) / Billing (ejecuta) | C-06 | [US-01] [PRD NFR-05] |
| **UC-05** | Consultar tracking tiempo real | Consumidor | C-05, C-07 (lectura) | [Brief §A.4] [PRD NFR-04] |

**Prohibido** crear UC-06+ salvo que el evaluador lo pida explícitamente.

---

## Regla de granularidad (obligatoria)

Crear **UC nuevo** solo si cambia:

1. **Actor primario**
2. **Objetivo de negocio**
3. **Aggregate principal** (Order vs Payment vs Delivery)
4. **Bounded context**

Si no cambia → **flujo alternativo (FA-xx.y)** del UC padre.

| Flujo | Tratamiento |
|-------|-------------|
| Cancelación consumidor | FA de UC-01 |
| Reasignación courier / back office | FA de UC-03 |
| Notificaciones | Side-effect; no UC (C-07) |
| Gestión de menús | Fuera del mínimo FSD [PRD §5.1] — no UC |

---

## Estados Order (usar en UCs)

`DRAFT` → `CONFIRMED` → `TICKET_ISSUED` → `PREPARING` → `READY` → `DELIVERY_ASSIGNED` → `IN_DELIVERY` → `DELIVERED` | `CANCELLED`

**Invariante [PRD NFR-07]:** no `DELIVERED` sin `READY` e `IN_DELIVERY`.

---

## Estructura obligatoria de salida (`docs/FSD.md`)

```markdown
# FSD — FTGO Platform
(tabla metadatos: versión, PRD upstream, origen)

## Introducción
## Tabla resumen de casos de uso
## Trazabilidad PRD → FSD
## Matriz NFR × UC

## UC-01: ...
(tabla: Actor, Capacidad PRD, Origen, BC, Aggregate)
#### Precondiciones
#### Flujo principal
#### Flujos alternativos
#### Postcondiciones
#### Given/When/Then
(escenario principal + escenario alterno si aplica)

(repetir UC-02 … UC-05)

## Diagrama de dependencias entre UCs
## Criterios de aceptación del FSD
## Referencias downstream (ADR, C4)
```

---

## Plantilla por UC (copiar estructura)

```markdown
## UC-0X: <Nombre>

| Campo | Valor |
| **Actor primario** | |
| **Capacidad PRD** | |
| **Origen** | |
| **Bounded context** | |
| **Aggregate** | |

#### Precondiciones
- ...

#### Flujo principal
1. ...

#### Flujos alternativos
**FA-0X.1 — ...**

#### Postcondiciones
- **Éxito:** ...
- **Fallo:** ...

#### Given/When/Then
**Escenario principal**
- **Given:** ...
- **When:** ...
- **Then:** ...

**Escenario alterno** (obligatorio en UC-01, UC-02, UC-03, UC-04, UC-05 según NFR)
- **Given:** ...
- **When:** ...
- **Then:** ...
```

### GWT obligatorios por UC

| UC | Escenario principal | Escenario alterno obligatorio |
|----|---------------------|------------------------------|
| UC-01 | Confirmar pedido | Pago pendiente Stripe [NFR-05] |
| UC-02 | Aceptar ticket | Rechazar ticket [US-02] |
| UC-03 | Aceptar entrega | Rechazar entrega [US-03] |
| UC-04 | Cobro exitoso | Stripe 5xx/timeout [NFR-05] |
| UC-05 | Tracking en entrega | Modo degradado Maps [NFR-04] |

---

## Anti-patterns (NO hacer)

| Anti-pattern | Corrección |
|--------------|------------|
| UC “Gestionar menú” | FA o fuera de alcance; C-02 no es UC mínimo |
| UC “Programa loyalty” | Dominio inventado |
| GWT genérico “el sistema funciona” | Estados y datos concretos (Order, TICKET_ISSUED) |
| Omitir FA cancelación en UC-01 | FA-01.4 obligatorio |
| UC separado por cada notificación | C-07 transversal |
| Duplicar PRD (copiar NFRs largos) | Referenciar `[PRD NFR-xx]` |
| Prescribir Kafka/REST en FSD | Remitir a ADR-0002 [PRD ASM-03] |
| Un solo GWT por UC sin alterno | Tabla GWT obligatorios arriba |
| Actor “Sistema” como primario sin consumidor/restaurant/courier | UC-04: Consumidor inicia; Billing ejecuta (nota en tabla) |

---

## Verification (ejecutar antes de entregar)

### Checklist

- [ ] Exactamente **5** secciones `## UC-01` … `## UC-05`
- [ ] Cada UC: tabla metadatos + pre + principal + FA + post + GWT
- [ ] **≥ 10** bloques GWT (5 principales + 5 alternos mínimo)
- [ ] Tabla resumen con columnas Actor, Capacidad, Origen, NFRs
- [ ] Matriz **NFR × UC** con ✓
- [ ] Trazabilidad PRD RF-01…03 → UC
- [ ] Introducción menciona Strangler [PRD NFR-09] sin diseñar infra
- [ ] Sin `UC-06` ni dominio inventado
- [ ] UC-04 coherente con [PRD NFR-05] (pedido no se pierde si Stripe cae)

### Puntuación mínima

| Ítem | Mínimo |
|------|--------|
| UCs | 5/5 |
| GWT (Given+When+Then) por UC | 5/5 |
| Escenarios alternos | 5/5 |
| Trazabilidad [US] o [PRD] por UC | 5/5 |

---

## Examples

### Ejemplo — Tabla resumen (fragmento)

```markdown
| UC | Nombre | Actor | Capacidad PRD | Origen |
| UC-01 | Tomar pedido | Consumidor | C-03 | [US-01] |
```

### Ejemplo — GWT UC-04 resiliencia

```markdown
- **Given:** un Order `CONFIRMED` listo para cobro
- **When:** Billing invoca Stripe y obtiene error 5xx o timeout
- **Then:** Payment queda `PAYMENT_PENDING`, el Order no se elimina y se programa reintento
```

### Ejemplo — FA cancelación (UC-01)

```markdown
**FA-01.4 — Cancelación consumidor:** antes de `TICKET_ISSUED`; Order → `CANCELLED`; reverso vía UC-04 si hubo captura.
```

### Ejemplo — Inválido (rechazar)

```markdown
## UC-06: Gestionar programa de puntos
```
→ **UC-06 no permitido** → eliminar.

---

## Instrucciones de ejecución

1. Leer `docs/PRD.md` completo.
2. Generar **solo** `docs/FSD.md` en español técnico.
3. Ejecutar **Verification**.
4. Incluir diagrama de dependencias UC-01 → UC-04, UC-02 → UC-03 → UC-05.
5. Referencias downstream: ADR-0001, ADR-0002, C4.

---

## Self-check final

| Pregunta | Requerido |
|----------|-----------|
| ¿5 UCs obligatorios? | Sí |
| ¿GWT en todos? | Sí |
| ¿Trazable a PRD/US? | Sí |
| ¿Granularidad respetada? | Sí |
| ¿Sin dominio inventado? | Sí |
| ¿Coherente NFR-07 estados? | Sí |

**Si No → corregir antes de entregar.**

---

*Fin del prompt ejecutable v2.0 — FSD FTGO*
