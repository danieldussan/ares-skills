---
name: ares-flujos
description: Usa esta skill cuando el usuario quiera crear, editar o revisar flujos del bot de WhatsApp a través del MCP, o escribir las instrucciones de la IA (nodo «Agente de IA», preguntas frecuentes, instrucciones del negocio). Explica el orden seguro de trabajo y qué va en cada capa de instrucciones.
---

# Flujos del bot por MCP

> Alcance: ayuda con cómo funciona el bot y cómo aprovechar sus herramientas. Las decisiones comerciales (plazos y urgencia de una oferta, precios, aportes, qué ofrecer) son del dueño del flujo: no las cuestiones ni las cambies; implementa la que te pidan.

Para saber qué hace cada nodo y cómo se conectan, usa la skill `ares-nodos`; para bots y conexión, `ares-bots`.

## Orden de trabajo (siempre)

1. `list_flows` y `get_flow` (incluye `settings` del flujo): entiende el flujo antes de tocarlo.
2. `describe_node_types`: llámala ANTES de armar un nodo; trae el `config` exacto de cada tipo y qué bloquea publicar.
3. `describe_prompt_layers`: llámala ANTES de escribir o corregir cualquier instrucción de IA.
4. `edit_flow` (o `create_draft` + `edit_draft`) para nodos y `update_flow_settings` para los ajustes del flujo: nunca edites producción directo; todo pasa por un borrador.
5. `validate_flow` y `simulate_flow`: valida y prueba. El simulador usa la IA real y gasta la clave de IA de la organización.
6. `diff_flow` y solo entonces `publish_flow`. Si `update_flow_settings` devolvió `driftFromLive` con campos, el borrador ya existía con valores distintos a los del flujo vivo: revísalo con `get_flow` del borrador y confírmalo con la persona antes de publicar. Si algo sale mal: `list_versions` y `restore_version`.
7. Ya publicado: `set_post_flow_ai` si la persona quiere que la IA hable solo dentro del nodo «Agente de IA» (`node_only`); actúa sobre el flujo vivo y falla si no hay nodo de IA conectado.

**Antes de crear nada, busca de dónde partir.** Con `list_flows` revisa si ya hay un flujo parecido (de la organización o una copia de uno que funciona). Si lo hay, `duplicate_flow` crea una copia independiente (nace inactiva, hereda el producto): edítala con `edit_flow` y publícala con `publish_flow`. Armar un flujo desde cero solo se justifica si no hay nada parecido; no rehagas lo que ya está hecho y probado.

**Datos de pago con botones de copiar.** Al escribir o adaptar el mensaje con los datos de pago, ponlos en una pieza `copy_buttons` (un botón por dato: cuenta, CLABE, alias, Yape, correo…), salvo que la persona pida explícitamente texto plano. Detalle en `ares-nodos` (nodo `message`).

Las tools de escritura requieren rol propietario o administrador.

## Las tres capas de instrucciones de la IA

Se suman, ninguna reemplaza a otra. Orden: organización → flujo → reglas fijas del sistema → nodo.

| Capa                | Pantalla                                           | Qué va                                                                                                   | Qué NO va                                                               |
| ------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Organización        | «Cómo habla tu IA» (Ajustes → IA)                  | Voz, trato, límites de lo que puede prometer                                                             | Productos, precios, reglas que el sistema ya impone                     |
| Flujo               | «Qué ofrece este flujo»                            | Solo lo general del producto: qué es, qué incluye, precio o rango y cómo se paga                         | La misión de un paso, el tono de marca, reglas que el sistema ya aplica |
| Nodo «Agente de IA» | «Qué debe lograr aquí» (`objective` + `faq_pairs`) | La misión del paso: etapa actual, qué hacer en cada caso; datos que cambian poco en preguntas frecuentes | Qué es el producto y su precio, reglas fijas, lógica de ramas           |

En el dashboard los campos se llaman «Qué ofrece este flujo» (configuración del flujo) y «Qué debe lograr aquí» (panel del nodo «Agente de IA»).

Las tres se pueden escribir por MCP:

| Capa         | Cómo                                                                                                                                                                                                                                                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Organización | `get_org_ai_instructions` → `set_org_ai_instructions` con `expectedPrevious`. Aplica YA a todos los flujos y no tiene borrador: muestra el texto exacto a la persona y espera su confirmación antes de llamarla. Guarda el `previous` que devuelve por si hay que revertir; si `changed` es false, no se escribió nada porque el texto ya era ese. |
| Flujo        | `update_flow_settings` con `systemPrompt` (escribe en el borrador) → `publish_flow`. `get_flow` devuelve `settings` con los valores actuales.                                                                                                                                                                                                      |
| Nodo         | `edit_flow` con `graphOps` sobre el nodo `ai_checkpoint` (`objective`, `faq_pairs`) → `publish_flow`.                                                                                                                                                                                                                                              |

Fuera de tu alcance: activar o desactivar el flujo, asignarlo a un bot y cambiar su producto. Dile a la persona que lo haga en el dashboard.

## Cómo escribir bien

- **Objetivo del nodo:** etapa + misión + casos. Ejemplo corto: «ETAPA: la persona ya recibió el material y el aporte es voluntario. MISIÓN: resolver dudas y, si quiere aportar, pedir el comprobante. Si no puede abrir el archivo, pregunta cuál y qué error ve.». Sin qué es el producto ni su precio: eso va en «Qué ofrece este flujo».
- **Preguntas frecuentes:** una fila por tema, «si preguntan por…» → «responde…», con el dato exacto: duración, garantías, qué incluye, cómo se recibe, qué hacer si no abre el archivo. Súmale una fila por cada pregunta real que ya llegó a una persona. La descripción general del producto sigue en «Qué ofrece este flujo».
- **No repitas reglas fijas.** El sistema ya fuerza tuteo, mensajes cortos, no decir que es una IA, no revelar herramientas y no inventar precios, fechas ni stock.
- **Si falta un dato**, la IA debe decir que no lo sabe; no lo completes con suposiciones. Pero no publiques con huecos: pregúntale el dato a la persona. Nunca escribas «no confirmado» en una capa; si de verdad se desconoce, escribe la frase exacta que debe decir la IA y a quién escala (soporte).
- **«Qué ofrece este flujo» es solo datos del producto.** No escribas ahí el orden de los pasos ni cómo se envía algo («en el paso de pago se envía el QR…»): la IA lo toma como propio y promete envíos que no puede hacer. Datos de pago sí van (cuenta, titular), pero lo que se envía lo envía un nodo o una utilidad.
- **Los avisos de `validate_flow` no bloquean pero cuentan.** Léelos y corrígelos o decide por qué no aplican antes de `publish_flow`.
- **Salidas:** las ramas se dibujan con las salidas del nodo, no se describen en el texto.
- **Límite de longitud:** `systemPrompt` e instrucciones de la organización aceptan hasta 10.000 caracteres por MCP; el dashboard recomienda hasta 1.500 (organización), 3.000 (flujo) y 5.000 (nodo).

## Qué hace el sistema por su cuenta

- No inyecta productos al prompt: los datos del negocio los escribe la persona, en el flujo o en las preguntas frecuentes.
- Si la IA no puede responder (clave rota, proveedor caído), el cliente no recibe texto automático y la conversación pasa a revisión humana.
- Un nodo «Agente de IA» sin la salida de tiempo agotado (`timeout`) conectada no se puede publicar, salvo que use espera sin límite.
- «Esperar respuesta» (`wait_reply`) no contesta nada, pero si su salida `next` lleva directo (sin pausa) a un «Agente de IA», ese agente contesta el mismo mensaje de la persona como su primer turno. Con pausa en esa conexión, el agente responde al siguiente mensaje.
- Al agotar el máximo de intercambios (`max_turns`, 2–20, por defecto 6), el nodo «Agente de IA» sigue por la salida `max_turns` («Demasiados intentos») si está conectada; si no, el cliente no recibe respuesta y la conversación pasa a revisión manual. Conéctala a un nodo de mensaje y luego a `handoff_agent` para avisar al cliente. `max_turns` y `timeout` son salidas del sistema: no las elige la IA.
- Modo del nodo «Agente de IA» (`ai_mode`): `conversational` (por defecto) deja que la IA converse con libertad dentro de sus instrucciones; `strict` («solo lo que escribí») la limita a lo que hay en las instrucciones, la misión y las preguntas frecuentes. Usa `strict` cuando un dato inventado sería grave (precios, plazos, garantías); usa `conversational` para charla de ventas abierta.
- Interruptor `can_request_support` (`true`/`false`): con `true` la IA puede pasar la conversación a soporte y sale por la salida de sistema `support` («Pide soporte»). Conéctala a un mensaje y luego a `handoff_agent`; sin conectar, la conversación pasa a revisión del equipo. La validación avisa (no es error) si el interruptor está encendido sin la salida, o si la salida existe con el interruptor apagado. Hacia el cliente se habla siempre de «soporte»: nunca de «humano», «persona» ni «bot».
- Nodo `wait_payment` con precios (`expected_prices`: una fila `{ currency, amount }` por moneda, sin conversión; la moneda que se exige a cada cliente es la que el sistema le resolvió por su país). Sin precios valida como siempre. `accept_higher` (por defecto `false`): con `true` un monto mayor también valida; uno menor siempre va a revisión.
- Con precios, un monto distinto, una moneda sin precio o un comprobante en otra moneda sale por `pending_manual_review` con el motivo escrito en la nota de la conversación y el aviso; el flujo sigue por la salida `amount_mismatch` («Monto distinto»), que solo existe con precios esperados válidos (o si ya tiene una conexión). Lo que conectes ahí es el aviso al cliente: el sistema manda el texto por defecto («Recibimos tu comprobante, pero el monto no coincide con lo esperado. Alguien de soporte lo revisará.») SOLO si esa salida no le escribe al cliente. Cuenta como que escribe si llega a `message`, `question`, `choice`, `carousel` o `capture` con algo que mandar, directo o tras `tag_action` / `capi_event`; tras una `condition`, solo si todas sus ramas escriben. Sin conectar o hacia un nodo mudo (`handoff_agent`, otro `wait_payment`, `ai_checkpoint`…), se manda. Tras la salida la conversación puede seguir en el flujo (`en_flujo` si lleva a una pregunta, a la IA o a otra espera de pago); el pago pendiente sigue en Pagos y en el aviso de Discord. Para que el cliente reintente, conecta la salida de vuelta a un `wait_payment`: si no, el comprobante corregido no tiene nodo que lo reciba y va a revisión manual. Con precios y la salida sin conectar, publicar avisa (warning, no error); conectada sin precios, también avisa. `pending_manual_review` queda para monto alto y posible duplicado (sin nodo activo).
- Falla cerrado: si no se puede identificar el `wait_payment` que cobra (espera perdida, nodo borrado) y algún `wait_payment` del flujo tiene precios, o si `expected_prices` está mal formado, el pago va a `pending_manual_review`.
- Varios nodos `wait_payment` en el mismo flujo permiten cobrar varios pagos (producto + upsell): un segundo comprobante solo cuenta como pago nuevo si hay un `wait_payment` activo; la entrega garantizada entrega lo que cuelga de la salida `validated` de cada nodo. Un pago sin nodo (manual) cuenta como del primer `wait_payment`.

## Flujo de venta bien armado

Estructura probada para vender por WhatsApp. Adáptala a la persona; no la rehagas desde cero si ya hay un flujo parecido (`duplicate_flow`).

1. **Abre corto y con una sola pregunta.** Un `message` de presentación (qué es, para quién, el beneficio) y enseguida `choice` o `question`. No encadenes más de 2 o 3 mensajes seguidos sin que el cliente responda.
2. **`question` para sí/no, `choice` para 3 o más opciones, `carousel` para mostrar planes con imagen.** Deja siempre la salida de «no entendí» (`unclear`) conectada a un mensaje que repita las opciones.
3. **Una rama para dudas.** «Tengo una duda» lleva a un `message` («¿Cuál es tu duda?») y luego al «Agente de IA» con salidas «listo» y «no interesado». Ver la lista de abajo.
4. **Oferta antes del pago.** Precio, qué incluye y qué no, en un mensaje corto y claro. Si el flujo cobra en monedas distintas, configura `expected_prices` en `wait_payment`.
5. **Datos de pago con botones de copiar.** El mensaje con los datos va en una pieza `copy_buttons` (un botón por dato: cuenta, CLABE, alias, Yape…), salvo que la persona pida texto plano. Después, `wait_payment`.
6. **Cierra cada salida de `wait_payment`.** `validated` → `message` de entrega (con `is_delivery`, para que se reenvíe a quien pagó y no lo recibió); `rejected` → pedir otro comprobante y volver al mismo `wait_payment`; `pending_manual_review` → aviso → `handoff_agent`; con precios, también `amount_mismatch`.
7. **Nunca dejes un callejón sin salida.** Toda salida de «no pude resolver» termina en `handoff_agent`. Las salidas `timeout`, `max_turns` y `support` de la IA también.
8. **Tono:** tuteo neutro, frases cortas, un emoji de apoyo como máximo por mensaje, sin párrafos largos. Usa variables como `{{contacto.nombre}}` solo si siempre existen.
9. **Eventos de Meta (`capi_event`)** después de que pasó lo valioso (lead calificado, pago validado), nunca al inicio.
10. **Mide y ajusta.** Prueba con `simulate_flow` cada salida y, en producción, revisa la IA con `ares-revision-ia`.

## Flujo completo: lista antes de publicar

`describe_prompt_layers` devuelve esta lista en `authoringChecklist`. Un flujo con IA no está listo hasta que cada punto se cumple o la persona dice que no aplica:

1. **Catálogo:** si el producto es una lista (calendarios, capítulos, recetas), el índice completo (número y nombre) va en «Qué ofrece este flujo», junto con lo que NO incluye. Sin él, ante «¿trae el de X?» la IA escala o improvisa.
2. **Preguntas frecuentes:** al menos 6 filas con datos reales (qué incluye, cómo se paga, cómo se recibe, qué hacer si no abre el archivo, cómo usarlo o imprimirlo, soporte).
3. **Datos desconocidos:** resueltos con la persona o con frase de escalación a soporte. Sin «no confirmado» en ninguna capa.
4. **Alcance:** lo que el producto NO es o no incluye, para que la IA no complete el vacío. `ai_mode: strict` si un dato inventado sería grave.
5. **Monto mínimo o sugerido:** qué hace la IA si la persona no llega (agradecer, aceptar menos sin prometer lo que depende del monto, no repetir el rango).
6. **Una sola fuente de verdad:** plazos, cantidades y promesas de los mensajes del flujo coinciden con las capas de IA (si el mensaje dice «en 10 minutos» y el prompt «sin vencimiento», la IA contradice al mensaje; el plazo en sí es decisión del dueño, solo hay que decirle a la IA qué responder si preguntan).
7. **Promesa = entrega:** si el primer mensaje promete 50 y el chat manda 5 más un enlace, dilo en ese mensaje.
8. **Entrega por enlace:** pide a la persona abrirlo en incógnito antes de publicar; debe abrir para cualquiera con el enlace. El MCP no puede verificarlo.
9. **Apertura de la rama «Tengo una duda»:** un `message` entre el `choice`/`question` y el «Agente de IA» («¿Cuál es tu duda?»); si no, la persona espera en silencio.
10. **Cortesías:** el objetivo pide una línea y cierre ante «gracias» o un emoji; `max_turns` de 6 o más.
11. **Países:** dos flujos con la misma frase de arranque y distinto país o moneda llevan `allowedCountries` en ambos (`update_flow_settings`).
12. **Datos de pago:** el mensaje que los muestra usa `copy_buttons` (un botón por dato), salvo que la persona haya pedido texto plano.
13. **Prueba:** `simulate_flow` con preguntas difíciles (catálogo, «no tengo para el monto», «no abre el archivo», un emoji).

Con el flujo ya en producción, la skill `ares-revision-ia` revisa cómo responde la IA con conversaciones reales.

## Antes de terminar

`validate_flow` sin errores bloqueantes, la lista de arriba repasada, `simulate_flow` probado con al menos un mensaje de cada salida, `diff_flow` revisado con la persona.

Para el nodo «Agente de IA» y las utilidades, usa la skill ares-nodos-ia.
