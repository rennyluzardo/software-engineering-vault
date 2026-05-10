# IMPLEMENTATION

## 1) Inicialización de PosGraphBuilder

### Patrón actual en NestJS
El entrypoint operativo está en el controlador, que construye `PosGraphBuilder` por request con dependencias inyectadas por Nest.

```ts
// backend/src/controllers/agent.controller.ts
@Controller('agent')
export class AgentController {
  constructor(
    private geminiAdapter: GeminiAdapterService,
    private inventoryService: InventoryService,
    private cartService: CartService,
  ) {}

  @Post('interact')
  async interact(@Body() dto: AgentInteractionDto): Promise<AgentState> {
    const graphBuilder = new PosGraphBuilder(
      this.geminiAdapter,
      this.inventoryService,
      this.cartService,
    );

    const initialState: AgentState = {
      messages: [{ role: 'user', content: dto.message }],
      currentStep: 'initial',
    };

    return await graphBuilder.invoke(initialState);
  }
}
```

### Configuración mínima de módulo
```ts
// backend/src/app.module.ts
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' })],
  controllers: [AgentController],
  providers: [GeminiAdapterService, InventoryService, CartService],
})
export class AppModule {}
```

## 2) Flujo técnico: "Quiero ver teclados"

### Paso a paso real
1. Input usuario llega a `POST /agent/interact`.
2. `analystNode` deja el estado en `routing`.
3. `routeByIntent` clasifica por keywords y envía a `inventoryToolNode`.
4. `inventoryToolNode` inyecta en `messages` un snapshot de inventario serializado.
5. `uiGeneratorNode`:
   - detecta contexto de inventario;
   - construye system prompt con componentes permitidos;
   - invoca modelo Gemini;
   - limpia markdown del JSON;
   - valida con Zod.
6. Backend responde `AgentState` con `uiSchema`.
7. Frontend (`ComponentRenderer`) resuelve el componente por registry y renderiza.

### Ejemplo de request
```bash
curl -X POST http://localhost:3000/agent/interact \
  -H "Content-Type: application/json" \
  -d '{"message":"Quiero ver teclados"}'
```

### Ejemplo de respuesta esperada (shape)
```json
{
  "messages": [
    { "role": "user", "content": "Quiero ver teclados" },
    { "role": "assistant", "content": "Productos disponibles: [...]" }
  ],
  "currentStep": "completed",
  "uiSchema": {
    "component": "ProductCatalog",
    "props": {
      "items": [
        { "id": 3, "name": "Teclado Mecánico RGB", "price": 79.99, "stock": 0 }
      ]
    },
    "status": "success"
  }
}
```

Nota: el sistema actual puede devolver catálogo amplio, no necesariamente filtrado exacto por intención semántica; depende del prompt y del contexto disponible.

## 3) Validación de stock en CartService

`addToCart()` protege invariantes de negocio antes de mutar estado:

```ts
addToCart(product: Product, quantity = 1) {
  if (product.stock <= 0) {
    return { success: false, message: `No hay stock disponible para ${product.name}` };
  }

  if (quantity > product.stock) {
    return {
      success: false,
      message: `Stock insuficiente. Solo hay ${product.stock} unidades disponibles de ${product.name}`,
    };
  }

  // merge con item existente + verificación de límite de stock
  // recalculateTotals()
}
```

### Edge cases importantes
- `stock = 0`: rechazo inmediato.
- `quantity > stock`: rechazo inmediato.
- `item existente + newQuantity > stock`: rechazo para evitar overbooking.
- `quantity = 0` o negativa: tests actuales validan que no agrega items, pero hoy devuelve `success: true`.

### Atomicidad local
La operación se ejecuta en memoria dentro del mismo servicio y request; no hay concurrencia transaccional distribuida todavía. Es atómica a nivel de estado interno del servicio singleton.

## 4) Knowledge Base: Problemas resueltos

### A) JSON Sanitization
Problema: el LLM puede envolver JSON en markdown.

Implementación actual:
```ts
private cleanJsonContent(rawContent: string): string {
  return rawContent.replace(/```json/g, '').replace(/```/g, '').trim();
}
```

Impacto: reduce fallos de parse por backticks y texto envolvente.

### B) MessageContent Normalization
Problema: LangChain puede entregar `content` como string o arreglo de bloques.

Implementación actual:
```ts
private normalizeMessageContent(content: unknown): string {
  if (typeof content === 'string') return content;
  if (Array.isArray(content)) {
    return content.map((c) => ('text' in c ? c.text : '')).join('');
  }
  return String(content);
}
```

Impacto: elimina errores por suposiciones de tipo en nodos/prompting.

### C) Zod Defaults en status
Problema: respuestas válidas sin `status` pueden romper contrato.

Implementación actual:
```ts
status: z.enum(['loading', 'success', 'error']).default('success')
```

Impacto: tolerancia controlada; evita rechazos innecesarios en casos benignos.

## 5) Testing Strategy

## Objetivo
Validar contrato y orquestación sin depender de latencia o variabilidad del LLM.

### A) Unit tests actuales (ya implementados)
- `backend/test/cart.service.spec.ts`: validación de stock y edge cases.
- `backend/test/integration/cart-flow.spec.ts`: flujo integrado inventario + carrito.

### B) Prueba de grafo con mock de LLM (recomendada)

```ts
import { PosGraphBuilder } from '../src/application/orchestration/pos-graph.builder';

it('genera UISchema válido sin llamar modelo real', async () => {
  const geminiAdapterMock = {
    getModel: () => ({
      invoke: async () => ({
        content: '{"component":"SimpleMessage","props":{"text":"ok","type":"info"},"status":"success"}',
      }),
    }),
  } as any;

  const graph = new PosGraphBuilder(geminiAdapterMock, inventoryService, cartService);
  const result = await graph.invoke({
    messages: [{ role: 'user', content: 'hola' }],
    currentStep: 'initial',
  });

  expect(result.uiSchema?.component).toBe('SimpleMessage');
  expect(result.uiSchema?.status).toBe('success');
});
```

### C) Validación de ComponentRenderer
- Caso feliz: componente registrado y render con props.
- Caso error: `uiSchema.component` no registrado y fallback visual.
- Caso de contrato: props mínimas por componente (`ProductCatalog`, `ShoppingCart`, `SimpleMessage`).

### D) Reglas de cobertura recomendadas
- Contrato Zod: casos positivos + negativos + payload malformado.
- uiGeneratorNode: respuesta con markdown, respuesta con JSON roto, respuesta con componente no esperado.
- routeByIntent: fixtures por intención y casos ambiguos ("mostrar"/"ver").
