# Adaptive-POS Architecture

## Overview
Adaptive-POS implementa un patrón de Generative UI donde el backend no entrega vistas estáticas, sino un contrato de UI serializable (UISchema) que el frontend interpreta dinámicamente. A diferencia de un POS tradicional, aquí la lógica de decisión de interfaz vive en un grafo de estados (LangGraph) que combina intención del usuario, estado del carrito y datos reales del inventario. El resultado es una arquitectura orientada a contexto: el mismo endpoint puede devolver catálogos, carrito o mensajes sin acoplar el frontend a reglas de negocio complejas.

El sistema actual opera con un flujo HTTP síncrono (`POST /agent/interact`) y no con streaming. El backend aplica validación Zod antes de retornar el esquema y usa fallbacks explícitos si el LLM responde JSON inválido. En frontend, un registry singleton resuelve el componente concreto por nombre. Este diseño reduce branching imperativo en React y concentra la evolución funcional en el contrato + orquestación.

## LangGraph Flow (Actual)

```text
User Input
   |
   v
[analystNode]
   |
   v  routeByIntent(state)
+-------------------------------+
| cart        -> [cartNode]     |
| inventory   -> [inventoryToolNode] |
| ui          -> [uiGeneratorNode]   |
+-------------------------------+
   |                |
   +-------+--------+
           v
     [uiGeneratorNode]
           |
           v
      validateUISchema(Zod)
           |
           v
       UISchema JSON
           |
           v
Frontend ComponentRenderer -> React Component
```

## Componentes Principales

### PosGraphBuilder
- Construye `StateGraph<AgentState>` con canales `messages`, `uiSchema`, `currentStep`.
- Registra nodos (`analystNode`, `cartNode`, `inventoryToolNode`, `uiGeneratorNode`) y define edges.
- Expone `invoke()` para ejecutar el flujo completo por request.

### analyzeIntent (analystNode + routeByIntent)
- `analystNode` prepara estado de routing sin mutar intención.
- `routeByIntent` usa heurísticas por keywords para clasificar en `cart`, `inventory` o `ui`.
- Estado pendiente: refactor para reducir ambigüedad de keywords compartidas como "ver" y "mostrar".

### inventoryNode (inventoryToolNode)
- Detecta consultas de inventario.
- Inyecta snapshot de productos en `messages` como contexto textual serializado.
- Fuente de datos: `InventoryService`.

### cartNode
- Interpreta acciones (`add`, `remove`, `clear`, `view`) desde texto libre.
- Busca producto en inventario y delega validación/mutación a `CartService`.
- Añade trazas de resultado al estado (`assistant` + `tool`) para consumo de `uiGeneratorNode`.

### uiGeneratorNode
- Compone prompt de sistema con componentes permitidos y contexto de datos reales.
- Normaliza contenidos multimodales (`normalizeMessageContent`) antes de invocar el modelo.
- Limpia markdown del JSON (`cleanJsonContent`) y valida con Zod.
- Si falla parse/validación, retorna fallback de error (`ErrorComponent`, `status: error`).

## Contrato de Datos (UISchema)

### Contrato Runtime (Backend Zod)
- `component: string` (no vacío)
- `props: Record<string, any>`
- `status: 'loading' | 'success' | 'error'` con default `success`

### Contrato de uso efectivo (Prompt + Frontend)
- Componentes operativos esperados: `ProductCatalog`, `ShoppingCart`, `SimpleMessage` (más `ErrorComponent` como fallback).

Por qué es crítico:
- Evita que JSON sintácticamente válido rompa rendering por estructura inválida.
- Permite fallar temprano en backend y entregar un estado controlado.
- Simplifica frontend: renderiza por contrato, no por parsing defensivo.

## ComponentRegistry Pattern
- Implementado como singleton (`registry`) en frontend.
- Patrón: `register(name, component)` + `get(name)`.
- `ComponentRenderer` resuelve `uiSchema.component` en runtime y pasa `props`.
- Si no existe mapping, devuelve bloque visual de error en UI.

Ventaja clave: extensión por composición. Agregar un nuevo componente exige registrar una entrada, sin rediseñar el renderer.

## InventoryService como SSOT
- Inventario en memoria (`Map<number, Product>`) inicializado con seed local.
- Todas las consultas de catálogo/stock y validaciones de carrito dependen de esta fuente.
- Implicaciones:
  - Consistencia local alta para MVP.
  - Sin persistencia cross-instance (riesgo al escalar horizontalmente).
  - Camino de evolución claro: reemplazar implementación interna por repositorio transaccional sin romper contratos de nodos/servicios.
