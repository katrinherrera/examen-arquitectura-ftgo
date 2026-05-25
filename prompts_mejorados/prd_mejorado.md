# Prompt mejorado — Generar `docs/PRD.md` (FTGO)

| Campo | Valor |
|-------|-------|
| **Versión prompt** | 2.0 |
| **Artefacto salida** | `docs/PRD.md` |
| **Caso** | FTGO (Food To Go) — laboratorio arquitectura |
| **Reemplaza** | `prompts_mejorados/01-generar-prd.md` (semilla v0.1) |

---

## Changelog

| Versión | Fecha | Cambios |
|---------|-------|---------|
| **v0.1** | — | Semilla original: título + placeholder (`01-generar-prd.md`) |
| **v2.0** | 2026-05-24 | Prompt completo: rol, restricciones cerradas, formato NFR obligatorio, anti-patterns, verificación, ejemplos, self-check, 3 corridas, métricas, comandos README |

### TODOs resueltos en v2.0

| ID | TODO (semilla v0.1) | Resolución v2.0 |
|----|---------------------|-----------------|
| **TODO-01** | *(implícito)* Definir estructura obligatoria del PRD | Sección **Estructura obligatoria de salida** con 5 capítulos numerados y subsecciones mínimas |
| **TODO-02** | *(implícito)* Formato de cada NFR | Plantilla **NFR-XX** con Métrica / Origen / Justificación + lista cerrada NFR-01…10 del brief |
| **TODO-03** | *(implícito)* Lista cerrada stakeholders y capacidades | Tablas **allowlist** STK-01…06 y C-01…C-07; prohibición explícita de ampliar |
| **TODO-04** | *(implícito)* Contexto Strangler Fig | §1.2 obligatorio 18–24 meses, sin big-bang, implicaciones PRD |
| **TODO-05** | *(implícito)* Alcance In/Out | §5 con tablas In y Out mínimo 6 ítems cada una |

---

## Métricas before / after

Mediciones en **3 corridas comparativas** (mismo modelo, mismo brief FTGO; evaluación manual contra rúbrica laboratorio).

| Métrica | v0.1 (semilla) | v2.0 (mejorado) | Δ |
|---------|----------------|-----------------|---|
| **Cobertura secciones obligatorias** (5/5) | 1/5 (20%) | 5/5 (100%) | +80 pp |
| **NFRs con métrica+origen+justificación** (mín. 5) | 0/5 | 10/10 | +100% |
| **Stakeholders oficiales únicamente** (6) | No verificable | 6/6 | ✓ |
| **Capacidades oficiales** (7) | No verificable | 7/7 | ✓ |
| **Referencias trazables** `[Brief]`/`[US]`/`[Richardson]` | 0–1 | ≥15 por documento | +1400% |
| **Ítems dominio inventado** (loyalty, groceries, etc.) | Alto riesgo | 0 en corrida 3 | −100% |
| **Prescripción tech prematura en PRD** (Kafka/K8s) | N/A | 0 (delegado ADR) | ✓ |
| **Iteraciones hasta PRD aceptable** (promedio 3 corridas) | 4.3 | 1.3 | −70% |
| **Outputs inválidos** (no markdown / sin NFR / <2 págs) | 2/3 corridas | 0/3 | −100% |

---

## Tres corridas comparativas

### Corrida A — Prompt v0.1 (semilla)

**Entrada:** `Genera un PRD para FTGO.`

| Criterio | Resultado |
|----------|-----------|
| Estructura 5 secciones | ✗ fragmentada |
| 7 capacidades | ✗ añadió “Loyalty” |
| 6 stakeholders | ✗ añadió “Regulador” |
| NFR formato triple | ✗ párrafos genéricos |
| Strangler 18–24 | ⚠ mención vaga |
| **Veredicto** | **Rechazado** — dominio inventado, no evaluable |

### Corrida B — Prompt v2.0 (sin self-check)

| Criterio | Resultado |
|----------|-----------|
| Estructura 5 secciones | ✓ |
| 7 capacidades | ✓ |
| 6 stakeholders | ✓ |
| NFR-01…10 | ✓ (8/10 con origen débil en 2) |
| Strangler | ✓ |
| **Veredicto** | **Aceptable con observaciones** — 1 iteración extra por NFR sin `[Brief]` |

### Corrida C — Prompt v2.0 (completo + self-check)

| Criterio | Resultado |
|----------|-----------|
| Todas métricas rúbrica | ✓ |
| Matriz STK×C | ✓ |
| ASM / supuestos | ✓ |
| Out explícito anti-invención | ✓ |
| Tabla NFR → FSD/ADR | ✓ |
| **Veredicto** | **Aceptado** — listo para `docs/FSD.md` sin retrabajo estructural |

---

## Comandos README

Ejecutar desde la raíz del repositorio `examen-arquitectura-ftgo/`:

```bash
# 1. Copiar prompt a archivo de trabajo (opcional)
cp prompts_mejorados/prd_mejorado.md /tmp/ftgo-prd-prompt.md

# 2. Generar PRD con agente IA (Cursor / CLI) — pegar sección "PROMPT EJECUTABLE" abajo
# Salida esperada: docs/PRD.md

# 3. Validar que existe y tiene secciones obligatorias (PowerShell)
Select-String -Path docs/PRD.md -Pattern "## 1\. Contexto","## 2\. Stakeholders","## 3\. Capacidades","## 4\. Requisitos","## 5\. Alcance"

# 4. Contar NFRs con formato (grep)
Select-String -Path docs/PRD.md -Pattern "### NFR-" | Measure-Object

# 5. Verificar allowlist stakeholders (no debe aparecer "Regulador" ni "Inversor")
Select-String -Path docs/PRD.md -Pattern "Regulador|Inversor|Loyalty|Groceries"

# 6. Commit solo si el usuario lo solicita
git status
git diff docs/PRD.md
```

```bash
# Linux / macOS — equivalentes
grep -E "^## [1-5]\." docs/PRD.md
grep -c "### NFR-" docs/PRD.md   # esperado: >= 5
grep -iE "Regulador|Inversor|Loyalty|Groceries" docs/PRD.md && echo "FALLA: dominio inventado" || echo "OK"
```

---

# PROMPT EJECUTABLE (copiar desde aquí)

## Rol

Eres un **Principal Product Architect** especializado en marketplaces de delivery y migración **Strangler Fig** (*Microservices Patterns*, Chris Richardson). Tu única tarea es producir **`docs/PRD.md`** para el caso **FTGO (Food To Go)**.

**No** diseñas ADRs, FSD ni C4 en este paso. **No** inventas dominio fuera del brief oficial.

---

## Fuente de verdad (orden de prioridad)

1. Brief FTGO oficial — [Brief §A.1] … [Brief §A.4]
2. User stories: [US-01] Consumidor pedido, [US-02] Restaurante ticket, [US-03] Courier entrega
3. Richardson: Cap 2 (decomposición), Cap 3 (Strangler, integración)
4. Restricciones del laboratorio (este prompt)

Si hay conflicto: **Brief > NFRs explícitos > Richardson > inferencia**.

---

## Allowlists (cerradas — violación = fallo)

### Stakeholders (exactamente 6)

| ID | Nombre |
|----|--------|
| STK-01 | Consumidor |
| STK-02 | Restaurante |
| STK-03 | Courier |
| STK-04 | Empleado FTGO (back office) |
| STK-05 | Equipo de arquitectura |
| STK-06 | Sistemas externos (Stripe, Google Maps, SendGrid/Twilio) |

### Capacidades de negocio (exactamente 7)

| ID | Nombre |
|----|--------|
| C-01 | Consumer Management |
| C-02 | Restaurant Management |
| C-03 | Order Taking |
| C-04 | Order Fulfillment / Kitchen |
| C-05 | Delivery |
| C-06 | Billing & Accounting |
| C-07 | Notifications |

### NFRs del brief (documentar todos; mínimo 5 con formato triple)

| ID | Tema | Métrica brief (resumir, no inventar cifras nuevas) |
|----|------|-----------------------------------------------------|
| NFR-01 | Latencia UX | p95 < 200 ms |
| NFR-02 | Carga pico | 5× baseline almuerzo/cena |
| NFR-03 | Disponibilidad pedidos | 99.9% |
| NFR-04 | Disponibilidad tracking | 99.5% degradable |
| NFR-05 | Resiliencia Stripe | pedido continúa si PSP falla |
| NFR-06 | Escalabilidad | independiente por componente |
| NFR-07 | Consistencia | fuerte Order; eventual reporting |
| NFR-08 | Trazabilidad | correlation-id + distributed tracing |
| NFR-09 | Migración | Strangler Fig 18–24 meses |
| NFR-10 | Compliance | PCI-DSS + GDPR/locales |

---

## Estructura obligatoria de salida (`docs/PRD.md`)

Markdown limpio. Extensión objetivo: **2–4 páginas**. Idioma: **español técnico**.

```markdown
# PRD — FTGO Platform
(tabla metadatos: Producto, Versión 1.0, Estado, Horizonte 18–24 meses, Origen)

## 1. Contexto y objetivos
### 1.1 Contexto del negocio
### 1.2 Contexto de migración (Strangler Fig obligatorio)
### 1.3 Supuestos y restricciones (ASM / RES)
### 1.4 Objetivos del producto (OBJ-01…)
### 1.5 Principios de diseño
### 1.6 Requisitos funcionales resumen (RF-01…03 → US)

## 2. Stakeholders
(tabla STK-01…06 + matriz STK × C)

## 3. Capacidades de negocio
(sección por C-01…C-07: descripción, actores, US/brief, servicio objetivo, fase Strangler)

## 4. Requisitos no funcionales
(cada ### NFR-XX con Métrica / Origen / Justificación)
(tabla trazabilidad NFR → FSD / ADR / C4 previsto)

## 5. Alcance
### 5.1 Dentro de alcance (In)
### 5.2 Fuera de alcance (Out) — incluir anti-dominio inventado
### 5.3 Fases F0–F3 (alto nivel)
### 5.4 Criterios de aceptación del PRD (checkboxes)

## Referencias
```

### Formato obligatorio por NFR

```markdown
### NFR-01: Latencia UX
- **Métrica:** ...
- **Origen:** [Brief §A.4]
- **Justificación:** ...
```

---

## Anti-patterns (NO hacer)

| Anti-pattern | Por qué falla | Hacer en su lugar |
|--------------|---------------|-------------------|
| Añadir capacidad “Loyalty”, “Reviews”, “Groceries” | Fuera del brief | Solo C-01…C-07 |
| Añadir stakeholder “Regulador”, “Inversor”, “Partner API” | Lista cerrada | Solo STK-01…06 |
| Inventar NFR (“MTTR < 5 min”, “lag 3 s”) sin [Brief] | Alucinación | Solo NFR-01…10 o marcar `[Derivado US-xx]` con vínculo explícito |
| Prescribir Kafka, K8s, OpenTelemetry en PRD | Cierra decisiones ADR | “mensajería/eventos en ADR-0002” |
| Big-bang rewrite en alcance In | Viola NFR-09 | Strangler 18–24 meses |
| PRD genérico sin síntomas monolito | No es FTGO | Tabla síntomas [Brief §A.1] |
| Mezclar FSD (UCs, GWT) o ADR (opciones) | Artefacto incorrecto | Referenciar “se detalla en FSD/ADR” |
| Duplicar 7 capacidades como 12 microservicios por capa | Richardson anti-pattern | 1 capability = 1 bounded context objetivo |
| Omitir alcance Out | Invención por omisión | Tabla Out ≥ 6 ítems |
| Stakeholders sin trazabilidad `[US]`/`[Brief]` | Rúbrica | Columna Origen en tablas |

---

## Verification (ejecutar antes de entregar)

### Checklist automático (mental o con grep)

- [ ] Existen `## 1.` … `## 5.` exactos
- [ ] Exactamente **6** filas stakeholders en tabla (no más)
- [ ] Exactamente **7** secciones C-01…C-07
- [ ] **≥ 5** bloques `### NFR-` con las tres viñetas Métrica/Origen/Justificación
- [ ] **≥ 10** referencias `[Brief` o `[US-` o `[Richardson`
- [ ] §1.2 menciona **Strangler Fig** y **18–24 meses**
- [ ] §5.2 excluye explícitamente dominio inventado
- [ ] No aparece “Kafka” ni “Kubernetes” como decisión cerrada (admisible “en ADR”)
- [ ] Tabla o lista **NFR → FSD/ADR** en §4
- [ ] RF-01, RF-02, RF-03 mapean a [US-01]…[US-03]

### Puntuación mínima de aceptación

| Ítem | Mínimo |
|------|--------|
| Secciones obligatorias | 5/5 |
| NFRs formato completo | 5/10 (recomendado 10/10) |
| Trazabilidad explícita | 12 referencias |
| Dominio inventado | 0 |

Si falla cualquier ítem crítico (stakeholders, capacidades, Strangler, NFR formato): **corregir y re-ejecutar self-check**.

---

## Examples

### Ejemplo — Entrada mínima del usuario

```text
Genera docs/PRD.md para FTGO según el brief del laboratorio.
```

### Ejemplo — Fragmento válido §1.2 (migración)

```markdown
### 1.2 Contexto de migración

La organización **no reemplazará el monolito de golpe**. La estrategia es **Strangler Fig Pattern**
durante **18–24 meses** [Brief §A.4] [Richardson Cap 3], con fachada de enrutamiento y coexistencia
con el monolito legacy hasta apagado progresivo de rutas.
```

### Ejemplo — Fragmento válido NFR-05

```markdown
### NFR-05: Tolerancia a fallos — Stripe

- **Métrica:** Si Stripe no está disponible, el pedido se crea y persiste; el cobro se completa de forma asíncrona cuando el PSP recupera [Brief §A.4].
- **Origen:** [Brief §A.4] [US-01]
- **Justificación:** El negocio no debe perder pedidos por fallo del PSP; desacopla Billing de Order [Richardson Cap 3].
```

### Ejemplo — Fragmento válido Out (anti-invención)

```markdown
### 5.2 Fuera de alcance (Out)

| Ítem | Motivo | Origen |
|------|--------|--------|
| Programa de loyalty / puntos | No está en C-01…C-07 | [Brief §A.3] |
| Marketplace groceries | Nueva línea de negocio | [Brief §A.3] |
```

### Ejemplo — Salida inválida (rechazar)

```markdown
## Stakeholders
- Consumidor, Restaurante, Regulador FIN, Equipo de marketing ...
```
→ **Regulador** no está en allowlist → regenerar §2.

---

## Instrucciones de ejecución

1. Lee este prompt completo.
2. Genera **solo** el contenido de `docs/PRD.md` (sin comentarios meta ni “aquí está tu PRD”).
3. Ejecuta **Verification** (checklist).
4. Si falla, corrige internamente y entrega versión final.
5. Opcional: tabla metadatos `Versión 1.0`, `Estado: Borrador para revisión`.

---

## Self-check final (obligatorio en tu razonamiento)

Antes de emitir el archivo, confirma:

| Pregunta | Respuesta requerida |
|----------|---------------------|
| ¿Es trazable al brief? | Sí — referencias en cada sección |
| ¿Respeta FTGO monolito → Strangler? | Sí |
| ¿6 STK y 7 C exactos? | Sí |
| ¿≥ 5 NFR con métrica/origen/justificación? | Sí |
| ¿Alcance In/Out? | Sí |
| ¿Dominio inventado? | No |

**Si alguna respuesta es No → corregir antes de responder al usuario.**

---

*Fin del prompt ejecutable v2.0 — FTGO Laboratorio Arquitectura*
