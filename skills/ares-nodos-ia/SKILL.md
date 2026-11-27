---
name: ares-nodos-ia
description: Usa esta skill cuando el usuario quiera armar, mejorar o revisar un nodo «Agente de IA» (ai_checkpoint) o las utilidades del flujo (envío de archivos, enlaces o datos de pago, con etiqueta al enviar). Explica el orden de trabajo por MCP, las reglas de calidad y recetas con ejemplos buenos y malos.
---

# Nodo «Agente de IA» y utilidades

> Alcance: ayuda con cómo funciona el bot y cómo aprovechar sus herramientas. Las decisiones comerciales (plazos y urgencia de una oferta, precios, aportes, qué ofrecer) son del dueño del flujo: no las cuestiones ni las cambies; implementa la que te pidan.

El nodo conversa varios turnos y sale por una de sus salidas. **Solo escribe texto: no adjunta archivos, enlaces ni datos de pago.** Eso sale por una utilidad o por un nodo de mensaje. Si la IA dice «te envío el PDF» y no hay utilidad, no llega nada.

Las pestañas del panel son «Conversación» («Misión», «Preguntas frecuentes», «Salidas»), «Acciones» («Utilidades», «Otras acciones») y «Seguridad» («Modo», «Límites», «Salidas de sistema»). El `config` exacto lo da `describe_node_types` con `type: "ai_checkpoint"`; el reparto de capas, `describe_prompt_layers`.

## Orden de trabajo

1. `describe_node_types` (`ai_checkpoint`, y `condition` si vas a controlar reenvíos) y `describe_prompt_layers`.
2. `get_flow`: revisa nodos, salidas y `utilities` ya existentes. Reutiliza antes de crear.
3. `upsert_utility` para cada envío que la IA deba poder hacer. **Escribe en el flujo vivo al instante**, no en el borrador: avisa a la persona.
4. `edit_flow` para la misión, las preguntas frecuentes, las salidas y el cableado.
5. `validate_flow` sin errores y **lee sus avisos**: no bloquean, pero cada uno señala un fallo real de construcción y trae el arreglo. `simulate_flow` con un mensaje por salida y con las pruebas difíciles (abajo); luego `publish_flow`.

Para el flujo completo (borrador, diff, versiones), usa la skill `ares-flujos`.

## Reglas de calidad

1. **Todo archivo, enlace o dato de pago sale por una utilidad.** Cada una con «¿Cuándo la usa la IA?» escrito como situación del cliente («Cuando pide el PDF o dice que no le llegó»), no como orden. La IA dispara una por turno y no tras un pago confirmado.
2. **Misión = ETAPA, MISIÓN, CASOS.** Producto y precio van en «Qué ofrece este flujo» (`update_flow_settings`, `systemPrompt`), no en la misión.
3. **Preguntas frecuentes = datos concretos, una fila por tema.**
4. **Modo.** «Solo lo que escribí» (`ai_mode: "strict"`) cuando un dato inventado sería grave, siempre con «Puede pedir soporte»; «Conversacional» para calificar o conversar una venta.
5. **Entregar una sola vez:** utilidad con «Al enviarla, marcar al contacto con» una etiqueta, y un nodo `condition` por esa etiqueta antes de volver a ofrecer el envío.
6. **Soporte por el interruptor «Puede pedir soporte»** (`can_request_support`, salida `support`), no con una salida propia. Conecta `support`, `timeout` y `max_turns` a un mensaje y luego a `handoff_agent`.
7. **`max_turns` ≥ 6.** Cada mensaje cuenta, también «gracias».
8. **Mensaje de apertura** si se llega desde «Tengo una duda».
9. **Probar antes de publicar:** «Probar nodo», «Ver lo que lee la IA» y `simulate_flow`.
10. **Cada archivo, QR o dato de pago tiene dueño.** O lo envía una utilidad que dispara la IA, o lo envía un nodo de mensaje del flujo. Nunca queda «en el prompt»: si «Qué ofrece este flujo» menciona un QR o una cuenta y ningún nodo ni utilidad lo envía, la IA dirá «aquí va el QR» y no llegará nada.
11. **«Qué ofrece este flujo» solo lleva datos del producto,** no el orden de los pasos del flujo («en el paso de pago se envía primero…»). La IA lee eso como instrucción propia y promete acciones que no ejecuta. El orden vive en los nodos.
12. **Antes de pasar a una persona, avisa.** `support`, `max_turns` y `timeout` van a un mensaje («Te conecto con alguien del equipo 💛») y luego a `handoff_agent`, con `message_for_agent` que diga de qué trataba la conversación.

## Salidas por utilidad: qué hacer cuando piden pagar

Una utilidad puede tener una salida (arista `utility:<id de la utilidad>` desde el nodo). La IA no la elige: la toma el sistema cuando la IA dispara esa utilidad.

- **El flujo entrega primero y cobra después** (aporte voluntario, muestra gratis): pedir pagar o el QR significa que quiere el producto. Misión: «si pregunta cómo pagar o pide el QR, di en una línea que el aporte es voluntario y que primero recibe la colección; ofrece enviársela». Cuando acepta («sí», «ok», «está bien»), sale por la salida de recibir. El QR llega luego por el nodo de pago normal. No des datos de pago a mitad del agente.
- **El flujo cobra antes:** utilidad «Datos de pago» (el QR y la cuenta) con su salida conectada a `capi_event` y `wait_payment`, para que el comprobante que llegue después tenga nodo de cobro. Si la utilidad tiene pasos con espera, pon el mismo retraso en la arista.
- **Entrega de archivos:** utilidad «Enviar archivos» con etiqueta y salida hacia el nodo que sigue («Entregado»).

## Pruebas difíciles antes de publicar

Con `simulate_flow`, un mensaje cada una, y revisa que la IA no prometa lo que no hizo: «¿cuánto cuesta?», «¿cómo pago?», «pásame el QR», «ok» o un emoji, «no me abre el enlace», una pregunta que no está en las preguntas frecuentes, y varias cortesías seguidas (para ver si agotan `max_turns`). Falla la prueba si la IA dice «aquí va», «te envío» o «ya te lo mandé» sin disparar una utilidad, o si escribe un paréntesis con «nota interna».

## Ejemplos

Misión mala: «Vendemos calendarios por US$12. Sé amable y ayuda.» (producto y precio en el lugar equivocado, sin etapa ni casos).

Misión buena:

```
ETAPA: ya vio la oferta de los calendarios.
MISIÓN: resolver dudas y llevarlo a pagar.
CASOS: si pide los archivos o dice que no le llegaron → envía la utilidad «Enviar los 5 calendarios».
Si dice que no le interesa → salida no_interesa. Si quiere pagar → salida listo.
```

`faq_pairs` malo: una fila «Información» con precio, duración y garantía juntos.

`faq_pairs` bueno: «¿Cuánto cuesta?» → «US$12, pago único»; «¿Cómo lo recibo?» → «Por este chat, en PDF»; «No me abre el PDF» → «Ábrelo con el lector de tu teléfono; si sigue igual te lo reenviamos».

## Recetas

- **Resolver dudas antes de comprar:** mensaje con la oferta → `ai_checkpoint` conversacional (4 o 5 preguntas frecuentes, salidas «listo» y «no_interesa») → esperar el pago.
- **Calificar en alto ticket:** `ai_checkpoint` conversacional, sin hablar de precio; la misión pide 2 o 3 preguntas y decide «calificado», «no_califica» o «persona».
- **Entrega y soporte de archivos:** utilidad «Enviar archivos» + pregunta frecuente «No me abre el PDF»; si tras reenviar sigue fallando, `support`.
- **Reenviar sin duplicar:** utilidad con etiqueta («Recibió calendarios») + `condition` por esa etiqueta antes del nodo de IA; si la tiene, mensaje «Ya te lo enviamos» y soporte.
- **Aporte voluntario sin presionar:** misión «agradecer y dar los datos solo si los pide»; los datos salen con la plantilla de utilidad «Dar datos de pago».

## Errores comunes

- Prometer en la misión un envío sin crear la utilidad.
- Poner precio o descripción del producto en la misión o en una sola pregunta frecuente gigante.
- `max_turns` de 3 o 4: se agota con una duda y un agradecimiento.
- Modo «Solo lo que escribí» sin «Puede pedir soporte»: la IA avisa que consulta, pero la conversación no pasa sola.
- Crear una salida propia para pedir ayuda en lugar de usar `support`.
- Dejar `timeout`, `max_turns` o `support` sin conectar.
- Confiar en que la IA «recuerde» que ya envió algo en vez de marcar con etiqueta.
- Escribir en «Qué ofrece este flujo» frases del motor («en el paso de pago se envía el QR…»): la IA las repite como si fueran suyas.
- Poner datos de pago en el prompt de un flujo cuyo agente queda antes del pago y no tiene utilidad de pago: la IA los menciona y promete enviarlos.
- Misión de una línea y menos de 6 preguntas frecuentes: la IA improvisa con lo que encuentre en el prompt largo.
- Mandar `max_turns` o `support` directo a `handoff_agent` sin mensaje: la persona se queda esperando.
- Ignorar los avisos de `validate_flow` porque no bloquean la publicación.
