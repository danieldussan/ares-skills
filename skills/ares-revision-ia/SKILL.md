---
name: ares-revision-ia
description: Usa esta skill cuando el usuario quiera revisar cómo está respondiendo la IA de un flujo con conversaciones reales (qué preguntas no supo contestar, qué inventó, cuándo escaló a una persona y por qué) o cuando haya quejas de clientes sobre las respuestas. Solo lee y propone; para aplicar los cambios usa la skill ares-flujos.
---

# Revisión de la IA de un flujo

Solo lectura: esta skill analiza y propone textos. No edita ni publica nada. Para aplicar los cambios, usa la skill `ares-flujos` y confirma cada texto con la persona.

## Qué necesitas

- El `flowId` del flujo (`list_flows`).
- Una muestra de conversaciones recientes: entre 20 y 30, de las últimas 48 a 96 horas.

## Pasos

1. **Entiende lo que la IA sabe.** `get_flow` (con `settings`), `get_org_ai_instructions` y `describe_prompt_layers`. Anota: el texto de «Qué ofrece este flujo», el `objective` y las `faq_pairs` de cada nodo `ai_checkpoint`, su `max_turns` y si tiene `ai_mode` estricto.
2. **Toma la muestra.** `get_conversations` con `flowId`. Separa tres grupos: las que terminaron en revisión de una persona (`revision_agente`, `revision_manual`), las que siguen con la IA (`en_ai`) y las que pagaron (`pago_confirmado`).
3. **Lee los mensajes.** `get_conversation_messages` por cada conversación; empieza por las del primer grupo. Cada mensaje trae `author`: `customer`, `flow`, `ai`, `human` o `system`.
4. **Clasifica cada respuesta de la IA** (`author: ai`):
   - **Correcta:** usa un dato escrito en las capas.
   - **Inventada:** afirma algo que no está escrito (alcance del producto, estilos, plazos, cifras).
   - **Contradice el flujo:** dice algo distinto de un mensaje del flujo (plazo, cantidad, si el aporte es obligatorio).
   - **Escalada evitable:** pasó a una persona una pregunta que un dato escrito habría resuelto.
   - **Escalada correcta:** el dato de verdad no existe o es un tema de soporte.
   - **Promesa sin cumplir:** dijo «te comparto…» y no llegó ningún archivo después.
   - **Relleno:** respondió a «gracias» o a un emoji gastando un turno.
5. **Mide la espera.** Para cada conversación en revisión, calcula cuánto pasó entre el último mensaje del cliente y la respuesta de `human`. Más de 2 horas en horario de atención se reporta.
6. **Agrupa por pregunta.** Cuenta cuántos clientes hicieron la misma pregunta que la IA no supo responder o respondió mal. Las más frecuentes pasan primero.
7. **Cruza con la lista de autoría.** Compara los hallazgos con `authoringChecklist` de `describe_prompt_layers` y corre `validate_flow` sobre el flujo.
8. **Propón los cambios.** Por cada problema, escribe el texto exacto para la capa correcta:
   - Dato nuevo y concreto → una fila de `faq_pairs` (disparador y respuesta).
   - Lista o alcance del producto → «Qué ofrece este flujo».
   - Qué hacer en un caso (monto bajo, no abre el archivo, cortesías) → `objective` del nodo.
   - Voz o límites generales → «Cómo habla tu IA».
   - Una contradicción entre un mensaje y el prompt → di cuál de los dos es la fuente correcta y pídele a la persona que lo decida.

## Qué entregar

Un informe corto:

1. **Resumen:** cuántas conversaciones revisaste y cuántas respuestas de cada clase.
2. **Problemas por frecuencia:** la pregunta o situación, cuántas veces aparece, un ejemplo literal del cliente y de la IA, y la causa (qué dato falta o se contradice).
3. **Cambios propuestos:** el texto listo para pegar y en qué capa va.
4. **Preguntas para la persona:** los datos que no existen en ninguna capa y que solo ella puede dar (por ejemplo, el índice del producto).

## Cuidado

- Los mensajes son de clientes reales: no copies teléfonos ni datos personales en el informe; usa el id de la conversación.
- Un dato que la IA no tiene no se inventa en la propuesta: pregúntalo.
- Las conversaciones simuladas no cuentan; ignora las que lo indiquen.

Para el nodo «Agente de IA» y las utilidades, usa la skill ares-nodos-ia.
