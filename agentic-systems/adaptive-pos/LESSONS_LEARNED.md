# LESSONS LEARNED

## 1) Qué funcionó bien

### Separación por nodos en LangGraph
Dividir en `analystNode`, `cartNode`, `inventoryToolNode` y `uiGeneratorNode` simplificó la responsabilidad de cada paso. Esta estructura facilita debugging, pruebas unitarias y evolución incremental.

### Zod como validador central
Mover la validación de contrato al backend evitó propagar errores ambiguos al frontend. El patrón parse + validate + fallback estabilizó el rendering.

### InventoryService como SSOT
Tener una única fuente de stock eliminó inconsistencias entre UI, carrito y narrativa del LLM. El carrito valida contra datos reales, no inferidos por prompt.

### Enfoque Spec-Driven Development
La disciplina de contrato (AGENTS + skills + schema) redujo drift entre agentes y mejoró trazabilidad técnica de cambios.

## 2) Qué no funcionó / dolores de cabeza

### Sanitización de JSON
Las respuestas del modelo pueden incluir backticks, encabezados markdown o ruido. Sin limpieza previa, `JSON.parse` falla de forma frecuente.

### Complejidad de MessageContent en LangChain
El campo `content` no siempre es string plano. Mezclar bloques multimodales rompe lógica si no se normaliza en un helper único.

### Orquestación de edges
El routing actual por keywords tiene ambigüedades (términos comunes como "ver" y "mostrar"). Esto explica el pendiente de refactor con `routeByIntent` más robusto y separación de lógica por intención.

## 3) Próximos pasos (Road Map)

1. Refactorizar edges de LangGraph con routing determinista por intención canónica.
2. Separar completamente lógica de `cartNode` e `inventoryToolNode` para minimizar heurísticas duplicadas.
3. Implementar mutaciones de stock en flujo de compra/pago (reserva, confirmación, rollback).
4. Sustituir inventario in-memory por persistencia transaccional en base de datos.
5. Añadir soporte multi-tenant con aislamiento por contexto de request.

## 4) Insights sobre Generative UI

### Cuándo es útil
- Flujos con alta variabilidad contextual (catálogo dinámico, recomendaciones, estados de carrito cambiantes).
- Dominios donde un contrato de UI serializable reduce complejidad del cliente.

### Cuándo no lo es
- Interfaces estables, repetitivas y con reglas de presentación totalmente previsibles.
- Experiencias donde latencia o costo de inferencia no se justifica.

### Costo/beneficio de mantener LLM en el loop
Beneficio: adaptabilidad y reducción de lógica condicional hardcodeada en frontend.
Costo: necesidad de gobernanza fuerte (contrato, tests adversativos, observabilidad, fallback robusto).

La conclusión operativa: Generative UI escala bien cuando hay contrato estricto, SSOT claro y un pipeline de validación defensivo como parte del camino crítico.
