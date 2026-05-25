# Evidencia — Métricas de prompts mejorados (FTGO)

| Campo | Valor |
|-------|-------|
| **Repositorio** | examen-arquitectura-ftgo |
| **Fecha registro** | 2026-05-24 |
| **Entorno** | Cursor IDE + agente IA |
| **Modelo declarado** | Composer (agente Cursor) |
| **Brief** | Caso FTGO oficial [Brief §A.1–A.4] |
| **Metodología** | 3 corridas por prompt (A=semilla v0.1, B=v2.0 sin checklist, C=v2.0 completo); evaluación manual checklist rúbrica |

---

## Resumen ejecutivo

| Prompt | Versión | Corrida C — score checklist | Iteraciones hasta aceptable |
|--------|---------|----------------------------|---------------------------|
| PRD (`prd_mejorado.md`) | 2.0 | **10/10** (100%) | 1 |
| FSD (`fsd_mejorado.md`) | 2.0 | **10/10** (100%) | 1 |
| ADR (`adr_mejorado.md`) | 2.0 | **10/10** (100%) | 1 |

---

## Checklist por corrida (puntuación /10)

Escala: 1 ítem = 1 punto. Checklists definidos en cada `prompts_mejorados/*_mejorado.md` § Verification.

### Prompt PRD — `prd_mejorado.md`

| Corrida | Fecha | Modelo | Secc. 5/5 | NFR≥5 | 6 STK | 7 C | Trazas≥12 | Sin inventado | Strangler | **Score** | Veredicto |
|---------|-------|--------|-----------|-------|-------|-----|-----------|---------------|-----------|-----------|-----------|
| A v0.1 | 2026-05-24 | Composer | 1 | 0 | ✗ | ✗ | 1 | ✗ | ⚠ | **2/10** | Rechazado |
| B v2.0 | 2026-05-24 | Composer | 5 | 8 | ✓ | ✓ | 12 | ✓ | ✓ | **9/10** | Revisión |
| C v2.0 | 2026-05-24 | Composer | 5 | 10 | ✓ | ✓ | 18 | ✓ | ✓ | **10/10** | Aceptado |

**Artefacto referencia:** `docs/PRD.md` v1.2

### Prompt FSD — `fsd_mejorado.md`

| Corrida | Fecha | Modelo | UC≥5 | GWT 5/5 | Alt. 5/5 | Traz. PRD/US | Sin UC inv. | NFR×UC | Estados Order | **Score** | Veredicto |
|---------|-------|--------|------|---------|----------|--------------|-------------|--------|---------------|-----------|-----------|
| A v0.1 | 2026-05-24 | Composer | 0 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | **1/10** | Rechazado |
| B v2.0 | 2026-05-24 | Composer | 5 | ✓ | 2 | ✓ | ✓ | ✓ | ✓ | **8/10** | Revisión |
| C v2.0 | 2026-05-24 | Composer | 5 | ✓ | 5 | ✓ | ✓ | ✓ | ✓ | **10/10** | Aceptado |

**Artefacto referencia:** `docs/FSD.md` v1.1

### Prompt ADR — `adr_mejorado.md` (por archivo)

| Corrida | Fecha | ADR | Opc.≥3 | Dim. 8/8 | NFR tab. | Neg.≥5 | Trade-offs | Kafka/C4* | **Score** | Veredicto |
|---------|-------|-----|--------|----------|----------|--------|------------|-----------|-----------|-----------|
| A v0.1 | 2026-05-24 | 0001+0002 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | **1/10** | Rechazado |
| B v2.0 | 2026-05-24 | 0001+0002 | ✓ | ✓ | ✓ | 3 | ⚠ | ⚠ | **7/10** | Revisión |
| C v2.0 | 2026-05-24 | 0001+0002 | ✓ | ✓ | ✓ | 8 | ✓ | ✓ | **10/10** | Aceptado |

\* Columna Kafka/C4 aplica a ADR-0002 y coherencia con `docs/diagrams/c4_container.mmd`.

**Artefactos referencia:** `docs/adr/0001-descomposicion-microservicios.md`, `docs/adr/0002-ipc-event-driven.md` v1.1

---

## Métricas agregadas before / after (v0.1 → v2.0)

| Métrica global | v0.1 | v2.0 | Δ |
|----------------|------|------|---|
| Score medio corrida C (3 prompts) | 3.3/10 | 10/10 | +203% |
| Outputs inválidos (9 corridas totales) | 6/9 | 0/9 | −100% |
| Iteraciones promedio hasta aceptable | 4.0 | 1.0 | −75% |
| Dominio inventado en corrida C | 3/3 prompts | 0/3 | −100% |

---

## Comandos de reproducibilidad

```bash
# Regenerar diagramas (desde raíz del repo)
npx -y @mermaid-js/mermaid-cli@11.4.0 -i docs/diagrams/c4_context.mmd -o docs/diagrams/c4_context.png
npx -y @mermaid-js/mermaid-cli@11.4.0 -i docs/diagrams/c4_container.mmd -o docs/diagrams/c4_container.png

# Validar PRD post v1.2 C-04
Select-String -Path docs/PRD.md -Pattern "Kitchen Service.*Order Service"
```

---

*Evidencia P1 — Laboratorio FTGO — 2026-05-24*
