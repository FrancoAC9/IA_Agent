# Checkpoint 1 · Agente Base y Motor de Razonamiento

**Curso:** Automatización Avanzada
**Alumno:** Franco Canova
**Entregable:** `checkpoint1_franco_canova.json`

Primera versión del proyecto integrador. Agente conversacional de calificación de
leads para una productora asesora de seguros, construido sobre n8n Cloud con el
nodo AI Agent en modo Tools Agent.

---

## Caso de negocio

El agente atiende el primer contacto de personas que consultan por un seguro.
Identifica el ramo, detecta el bien a asegurar y deriva el caso a la asesora
humana por correo, sin cotizar ni confirmar coberturas.

El criterio de derivación se definió a partir del comportamiento real del canal:
la mayoría de los interesados escribe con lo mínimo ("cuánto sale asegurar una
Suran 2013") y abandona si se le piden varios datos antes de atenderlo. Por eso
el umbral de activación de la herramienta es **un bien concreto a asegurar**, no
un conjunto completo de datos de contacto. El teléfono se pide después de
derivar, no antes.

---

## Arquitectura

```
[Chat Trigger]
      │
      ▼
[AI Agent · Tools Agent] ──┬── ai_languageModel → Anthropic Chat Model
      │                    ├── ai_memory        → Simple Memory (Session ID)
      │                    └── ai_tool          → notificar_asesor_humano
      ▼
[Code: formatear log]
      │
      ▼
[Gmail: reporte de observabilidad]
```

La distinción que organiza el diseño:

- **Chat Model y Tool** cuelgan lateralmente del agente. Son capacidades
  que el agente decide usar o no: autonomía probabilística del ciclo ReAct.
- **Code y Gmail final** cuelgan en `main`. Corren siempre, en orden fijo:
  automatización lineal determinista.

La herramienta nunca se conectó como nodo de acción secuencial del lienzo, que
es lo que la convertiría en un paso obligatorio y anularía la decisión del
agente.

---

## Componentes

| Requisito | Implementación |
|---|---|
| Trigger | Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) |
| Cerebro | AI Agent en modo `toolsAgent` |
| Modelo | Anthropic Chat Model · `claude-sonnet-5` · temperature 0 |
| Guardrail | `maxIterations: 8` |
| System Prompt | Modular: Rol → Regla crítica → Qué hacés → Cuándo derivás → Lectura del resultado → Límites → Estilo |
| Tool | Gmail Tool `notificar_asesor_humano`, conexión `ai_tool` |
| Observabilidad | `returnIntermediateSteps: true` → nodo Code → Gmail |


### Observabilidad

El nodo Code lee `intermediateSteps` de la salida del agente y arma un reporte en
texto plano con la traza completa: qué herramienta se activó, con qué entrada y
qué devolvió. Ese reporte se envía por Gmail al finalizar cada ejecución.

No es un log de la respuesta final: es la auditoría del recorrido de decisión,
que es lo que permite detectar comportamientos que no se ven desde el chat.

---

## Decisiones técnicas documentadas

**Modelo.** La consigna sugiere GPT-4o o Claude 3.5 Sonnet. `claude-3-5-sonnet-20241022`
fue deprecado el 13/08/2025 y retirado el 28/10/2025; las llamadas devuelven 404.
Se usó `claude-sonnet-5`.

**Credenciales.** La primera ejecución falló con
`Gateway credits eligibility check failed: nodeNotCovered`, un bug conocido de
n8n Cloud por el que el chequeo de créditos del gateway se ejecuta contra el nodo
padre en lugar del sub-nodo del modelo. Se resolvió usando una API key propia de
Anthropic en lugar de los créditos gratuitos de la plataforma.

---

## Cómo probar

1. Importar el `.json` en n8n.
2. Asignar credenciales: Anthropic API y Gmail OAuth2.
3. Reemplazar las direcciones de correo en el nodo `notificar_asesor_humano`
   (destinatario de la derivación) y en el nodo Gmail final (destinatario del
   reporte de observabilidad).
4. Abrir el chat y enviar: `Hola, cuánto sale asegurar una Suran Trendline 2013`.

Resultado esperado: el agente deriva en la primera respuesta, llega el correo al
asesor con el resumen estructurado, y llega el reporte de observabilidad con
`Iteraciones con herramienta: 1` y la traza de la llamada.

---

## Próximos hitos

| Módulo | Ampliación |
|---|---|
| M2 | Multi-agente · sub-workflow |
| M3 | Memoria persistente en Airtable por Session ID |
| M4 | Canal real de entrada |
| M5 | RAG sobre base documental de condiciones de póliza |
| M6 | Voz |
