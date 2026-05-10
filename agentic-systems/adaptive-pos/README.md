# Adaptive-POS: Generative UI System with LangGraph

## Overview
Sistema de Punto de Venta donde la UI se genera dinámicamente basada en intención del usuario e inventario real. Demuestra orquestación agentic con LangGraph, políticas zero-hallucination y Spec-Driven Development para mantener consistencia entre backend y frontend.

## Documentos
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Diagramas, flujo LangGraph y componentes.
- [DECISIONS.md](./DECISIONS.md) - ADRs, racional técnico y trade-offs.
- [IMPLEMENTATION.md](./IMPLEMENTATION.md) - Guía técnica, patrones y testing.
- [LESSONS_LEARNED.md](./LESSONS_LEARNED.md) - Hallazgos, riesgos y road map.

## Stack
NestJS + LangGraph.js | React 19 + Vite + Tailwind 4

## Key Concepts
- **Generative UI:** Interfaces que se adaptan en tiempo real.
- **Agentic Workflows:** LangGraph para orquestación multi-paso.
- **Zero-Hallucination:** Validación contractual y grounding con datos reales.
- **SSOT:** InventoryService como fuente única de verdad del stock.

## Caso de Uso
Ideal para sistemas donde:
- La UI depende de datos contextuales (inventario, carrito, estado operacional).
- Existen múltiples permutaciones de estado difíciles de codificar con if/else en frontend.
- Se busca reducir boilerplate visual manteniendo control estricto de contrato.
