# DECISIONS - Architectural Decision Records

## ADR-001: UISchema Contract with Zod

- Estado: Aceptado
- Fecha: 2026-05-10

### Problema
El LLM puede devolver JSON sintácticamente válido pero contractualmente incorrecto: componentes inexistentes, props incompatibles o `status` inválido. Sin validación robusta, el error se traslada al frontend.

### Decisión
Validar toda salida de UI en backend con `UISchemaSchema` (Zod) antes de responder. El parser aplica:
1. Sanitización de markdown (`cleanJsonContent`).
2. `JSON.parse`.
3. `validateUISchema`.

Si falla cualquier paso, se retorna fallback de error controlado.

### Trade-offs
- Costo: overhead mínimo de validación y manejo de errores en backend.
- Beneficio: contrato estable, fallas detectadas temprano y frontend desacoplado de validación estructural.

### Consecuencias
- El frontend puede asumir un `uiSchema` coherente y enfocarse en render.
- Las regresiones de prompt se detectan como errores de contrato y no como crashes de UI.
- Se habilita testing determinista del contrato.

---

## ADR-002: ComponentRegistry Singleton Pattern

- Estado: Aceptado
- Fecha: 2026-05-10

### Problema
El backend retorna nombres de componentes como strings. React necesita mapearlos a implementaciones reales de forma centralizada y extensible.

### Decisión
Usar un singleton `registry` en frontend con API explícita:
- `register(name, component)`
- `get(name)`
- `has(name)`
- `listRegistered()`

`ComponentRenderer` consume el registry y renderiza dinámicamente.

### Trade-offs
- Costo: punto central único (riesgo de dependencia global).
- Beneficio: fuente de verdad única para mapping, baja complejidad operativa y onboarding más rápido.

### Consecuencias
- Alta extensibilidad: nuevos componentes se agregan registrando una entrada.
- Fallback uniforme cuando un componente no existe.
- Menor probabilidad de divergencia entre contrato y render.

---

## ADR-003: InventoryService como Single Source of Truth

- Estado: Aceptado
- Fecha: 2026-05-10

### Problema
El stock no puede derivarse del prompt ni del frontend; requiere una autoridad de datos consistente para evitar alucinación funcional y errores de compra.

### Decisión
Centralizar lectura y mutación de stock en `InventoryService`. Todos los nodos y servicios dependientes (incluyendo `CartService`) consultan esta fuente.

### Trade-offs
- Costo actual: implementación in-memory no persistente.
- Beneficio inmediato: consistencia local, baja latencia y simplicidad de MVP.

### Escalabilidad
Evolución prevista a repositorio persistente (SQL/NoSQL) manteniendo la misma interfaz de servicio para preservar contratos de orquestación.

### Consecuencias
- Cart validation coherente con estado real de stock en runtime.
- Menor acoplamiento entre LLM y datos críticos.
- Path de migración controlado a infraestructura productiva.

---

## ADR-004: Spec-Driven Development (SDD) con AGENTS.md

- Estado: Aceptado
- Fecha: 2026-05-10

### Problema
Sin restricciones explícitas, agentes y prompts tienden a divergir: features inventadas, incumplimiento de contratos y cambios no coordinados entre backend/frontend.

### Decisión
Adoptar SDD con `AGENTS.md` como contrato operativo de roles, responsabilidades y políticas (incluyendo zero-hallucination). Complementar con skills específicas en `.agentic/skills/`.

### Mecanismo
- Definir/ajustar spec primero (Zod y tipos).
- Validar con QA (casos positivos, negativos y adversativos).
- Implementar backend/frontend alineados al contrato.
- Actualizar documentación y pruebas de regresión de prompt.

### Trade-offs
- Costo: mayor disciplina de proceso y documentación.
- Beneficio: menos incertidumbre sistémica y menor riesgo de regresión silenciosa.

### Resultado
- Zero-hallucination policy operable, no solo declarativa.
- Cambios más auditables y replicables para equipos multiagente.
