# FTGO — Laboratorio de Arquitectura

Migración evolutiva del monolito **Food To Go** hacia microservicios (Strangler Fig, 18–24 meses).

## Estructura del repositorio

```
examen-arquitectura-ftgo/
├── README.md
├── docs/
│   ├── PRD.md
│   ├── FSD.md
│   ├── adr/
│   │   ├── 0001-descomposicion-microservicios.md
│   │   └── 0002-event-driven-order-communication.md
│   └── diagrams/
│       ├── c4_context.mmd
│       └── c4_container.mmd
└── prompts_mejorados/
    ├── 01-generar-prd.md
    └── 02-generar-fsd.md
```

## Artefactos

| Ruta | Descripción |
|------|-------------|
| `docs/PRD.md` | Product Requirements: stakeholders, capacidades, NFRs, alcance |
| `docs/FSD.md` | Functional Spec: casos de uso con Given/When/Then |
| `docs/adr/0001-*.md` | ADR: estrategia Strangler Fig y descomposición |
| `docs/adr/0002-*.md` | ADR: comunicación, eventos y consistencia Order |
| `docs/diagrams/c4_context.mmd` | C4 Nivel 1 — Context |
| `docs/diagrams/c4_container.mmd` | C4 Nivel 2 — Container |
| `prompts_mejorados/*.md` | Prompts refinados con métricas y changelog |

## Trazabilidad

PRD → FSD → ADR → C4. Referencias: `[Brief]`, `[US-xx]`, `[PRD NFR-xx]`, `[Richardson Cap x]`.

## Comandos útiles

```bash
# Validar sintaxis Mermaid C4 (requiere mermaid-cli instalado)
npx -y @mermaid-js/mermaid-cli -i docs/diagrams/c4_context.mmd -o docs/diagrams/c4_context.png
npx -y @mermaid-js/mermaid-cli -i docs/diagrams/c4_container.mmd -o docs/diagrams/c4_container.png
```
