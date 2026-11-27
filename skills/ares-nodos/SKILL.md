---
name: ares-nodos
description: Usa esta skill cuando el usuario quiera crear o armar un flujo del bot de WhatsApp y necesites saber qué hace cada tipo de nodo, cuándo usar uno u otro, qué salidas tiene y cómo conectarlos por MCP. Incluye recetas de flujos típicos (venta con comprobante, calificar con IA, reabrir con plantilla).
---

# Nodos de un flujo: qué hace cada uno

Esta skill explica el mapa. El `config` exacto de cada nodo (campos, tipos, obligatorios, ejemplo válido y qué bloquea publicar) lo da `describe_node_types`: llámala con `type` ANTES de armar cada nodo. El orden seguro de trabajo (borrador, validar, simular, publicar) está en la skill `ares-flujos`; las instrucciones de la IA, en `describe_prompt_layers`.

## Cómo es un flujo

Un flujo es un grafo: **nodos** unidos por **conexiones**. Cada conexión sale de una **salida** (handle) de un nodo y llega a otro nodo. La persona recorre el grafo de un nodo a otro; algunos nodos envían, otros esperan, otros deciden.

- Un nodo con una salida sin conectar es un hueco: el recorrido se corta ahí. `validate_flow` avisa o bloquea según el nodo.
- Hay nodos que esperan a la persona (pausan el recorrido) y nodos que siguen solos.
- Las conexiones pueden llevar una demora en segundos (`delaySeconds`).

## Resumen por familias

| Familia             | Nodos                                    | Qué hacen                                |
| ------------------- | ---------------------------------------- | ---------------------------------------- |
| Enviar              | `message`, `remarketing_template`        | Mandan contenido o una plantilla         |
| Preguntar con ramas | `question`, `choice`, `carousel`         | Preguntan y siguen por la respuesta      |
| Recoger datos       | `capture`                                | Piden datos y los guardan en `var.*`     |
| Esperar texto libre | `wait_reply`                             | Espera lo que escriba la persona         |
| IA                  | `ai_checkpoint`, `ai_generate`           | La IA conversa o genera texto e imágenes |
| Decidir sin IA      | `condition`                              | Bifurca por etiquetas, etapa o variables |
| Efectos             | `tag_action`, `capi_event`               | Etiquetan o reportan a Meta, y siguen    |
| Cobro               | `wait_payment`                           | Esperan un comprobante y lo validan      |
| Cierre              | `handoff_agent`                          | Pasan la conversación al equipo          |

## Los nodos, uno por uno

### `message` — enviar contenido

- **Hace:** envía una o más piezas (texto, imagen, documento, video, audio, botones de copiar) y sigue sola por `next`. No espera nada.
- **Úsalo para:** contenido puro. Si necesitas una respuesta, usa `question`, `choice` o una espera.
- **Salidas:** `next`.
- **Datos de pago: usa botones de copiar por defecto.** Cuenta, CLABE, alias, número de Yape/Plin, correo o cualquier dato que el cliente deba copiar va en una pieza `message_type: "copy_buttons"` (hasta 4 botones, un dato por botón, título ≤20 caracteres, el dato en `value`; el cuerpo va en `text_content`). El cliente copia con un toque y no se equivoca al transcribir. Solo déjalos como texto plano si la persona pide explícitamente que no haya botones; en Meta Cloud bajan solos a texto.
- **Detalles útiles:** el texto admite variables como `{{contacto.nombre}}` o `{{pago.ultimo.monto}}`; cada pieza puede tener variantes de texto y una pausa; la opción de entrega garantizada (`is_delivery`) hace que el contenido se reenvíe a quien pague y no lo haya recibido.

### `question` — pregunta de sí o no

- **Hace:** manda dos botones (sí / no). La respuesta, por botón o texto libre clasificado por IA, sigue por `yes` o `no`.
- **Úsalo para:** exactamente dos opciones. Para más, usa `choice`.
- **Salidas:** `yes`, `no`; opcionales `unclear` (no se entendió) y `no_response` (si fijas un tiempo de espera, debe estar conectada).
- **Ojo:** puede esperar sin límite (`wait_mode: unlimited`), pero eso no se combina con un tiempo límite.

### `choice` — opción múltiple

- **Hace:** pregunta con 2 a 10 opciones (botones hasta 3; lista o botones partidos de 4 a 10). Sigue por la salida cuyo id es la opción elegida.
- **Úsalo para:** alternativas de negocio («plan mensual», «plan anual», «no me interesa»).
- **Salidas:** una por opción (el id es el slug de la etiqueta), `unclear` (siempre debe estar conectada) y `no_response` (si fijas tiempo de espera).

### Media en los nodos

- La media de un nodo (imagen, audio, video, documento) debe vivir en la galería de la organización.
- Archivo local: usa `request_media_upload` (`filename`, `mimeType`, `sizeBytes`), súbelo con el `curl -T '<ruta>' -H 'Content-Type: <mime>' '<uploadUrl>'` que devuelve (Content-Type idéntico al declarado; la URL vale 10 min) y confirma con `confirm_media_upload` (`uploadId`). Los bytes no pasan por el chat, así que se conserva la calidad. Tope 50 MB (imagen 10 MB, audio 16 MB).
- URL https pública: `upload_media` con `sourceUrl`. Nunca pegues el archivo en base64: ya no existe.
- Pon el `publicUrl` resultante en `media_url`. Si no tienes shell/curl ni acceso al archivo, dile al usuario que lo suba a mano en https://dashboard.aresbot.app/media.
- No uses URLs de hosting externo: pueden cortarse y el mensaje dejaría de enviarse.

### `wait_reply` — esperar cualquier texto

- **Hace:** espera a que la persona escriba algo; cualquier texto cuenta como «ya contestó» y sigue por `next`. No usa IA ni clasifica.
- **Úsalo para:** pausas tipo «avísame cuando estés listo».
- **Salidas:** `next`; `no_response` solo si fijas un tiempo para rendirse (`giveup_delay_minutes`). Puede mandar un recordatorio.
- **Ojo:** si `next` lleva directo (sin demora) a un `ai_checkpoint`, ese texto es el primer turno de la IA y contesta el mismo mensaje.

### `carousel` — carrusel de tarjetas

- **Hace:** manda un carrusel de 1 a 10 tarjetas (imagen https + texto + hasta 3 botones cada una). Sigue por la salida cuyo id es el botón `reply` que la persona pulsó.
- **Úsalo para:** mostrar varios productos o planes con imagen y dejar elegir uno. Para opciones sin imagen, `choice`; para solo informar, `message`.
- **Salidas:** una por cada botón `reply` (su id), `unclear` («Texto libre», SIEMPRE conectada: bloquea publicar) y `no_response` (si fijas tiempo de espera, debe estar conectada). Los botones `url`, `copy` y `call` no son salidas.
- **Ojo:** si la persona escribe en vez de pulsar, se compara con las etiquetas `reply` (sin tildes ni mayúsculas) y, si no hay coincidencia única, la IA de la organización intenta asociarlo (`ai_text_matching`); si no, sale por `unclear`. Las etiquetas `reply` repetidas hacen que decida la IA. En Meta Cloud llega una versión simple (imágenes con pie de texto). Etiquetas de botón de hasta 20 caracteres; ids de botón únicos en el nodo.

### `capture` — capturar datos

- **Hace:** pide uno o varios datos (de 1 a 10 campos: texto, número, teléfono, correo, fecha u opción), los valida y los guarda como variables `var.<nombre>`. Esas variables se usan en `condition` («Variables») y como `{{var.nombre}}` en `message`, `question`, `choice` y `wait_reply`.
- **Modos:** `one_by_one` (por defecto): un campo a la vez, sin IA. `ai_extract`: una pregunta abierta y la IA de la organización saca los campos del texto; lo que falta se pide después (si no hay clave de IA, cae a uno por uno).
- **Úsalo para:** nombre, teléfono, ciudad, correo, fecha o una opción que vas a usar más adelante. Para esperar texto libre sin guardar nada, `wait_reply`; para conversar con herramientas, `ai_checkpoint`.
- **Salidas:** `next` (todo lo obligatorio guardado), `invalid` (intentos agotados; OBLIGATORIA si hay un campo obligatorio: sin ella no se publica) y `no_response` (solo si fijas `giveup_delay_minutes`).
- **Ojo:** una nota de voz cuenta como texto; una imagen o un botón no. Un dato sin valor deja el `{{var.x}}` vacío y hace falsa la condición que lo use. Se puede reintentar de 0 a 5 veces (`max_retries`).

### `ai_checkpoint` — «Agente de IA»

- **Hace:** cede el turno a la IA durante varios intercambios, con herramientas (utilidades, etiquetas, preguntas frecuentes, pedir soporte), hasta que resuelve el nodo hacia una salida.
- **Úsalo para:** conversación abierta donde la IA decide qué hacer varios turnos. Para recoger y contestar un dato puntual sin ramas, `capture`; si no necesitas respuesta generada, `wait_reply`.
- **Salidas:** las que tú defines como resultados del nodo (cada una con su id y rótulo; la IA elige una), más salidas del sistema: `timeout` (siempre debe estar conectada, salvo espera sin límite), `max_turns` («Demasiados intentos», al agotar los intercambios) y `support` (si activas «Puede pedir soporte»). Las del sistema no las elige la IA.
- **Ajustes clave:** `max_turns` (2 a 20, por defecto 6), `ai_mode` (`conversational` o `strict`: «solo lo que escribí»), `can_request_support`, `objective` y `faq_pairs` (ver `describe_prompt_layers`).
- **Salida por utilidad (opcional):** una arista `utility:<id de la utilidad>` saca la conversación del nodo cuando la IA dispara esa utilidad (p. ej. datos de pago → `capi_event` → `wait_payment`). Sin arista no cambia nada. La IA no la elige como salida.
- **Patrón seguro:** conecta `max_turns` y `support` a un `message` que avise al cliente y luego a `handoff_agent`. Sin conectar, la conversación pasa a revisión del equipo sin avisar al cliente.

### `ai_generate` — generar texto o imagen con IA

- **Hace:** genera contenido a partir de un prompt con variables, lo envía a la persona y sigue por su salida. Con `mode: "text"` responde en el mismo turno; con `mode: "image"` encola la generación y la conversación queda detenida en el nodo hasta que llega el resultado o vence el tiempo.
- **Úsalo para:** personalizar un mensaje con datos capturados (rutina, saludo, resumen) o producir un material visual, por ejemplo a partir de una foto que la persona mandó antes (`capture` con campo `image`). No es una conversación: para conversar usa `ai_checkpoint`.
- **Salidas:** `ok` y `error` en texto; `ok`, `error` y `timeout` en imagen. Conecta siempre `error` (y `timeout` en imagen): sin conectar, la conversación pasa a revisión del equipo con una nota del motivo y la persona nunca recibe texto crudo del proveedor.
- **Ajustes clave:** `mode`, `prompt` (1-4000 caracteres, obligatorio salvo que uses `shots`; admite `{{var.*}}`), `result_var` (guarda el texto o la URL en `var.<nombre>`), `send_result` (default true) y, en texto, `context_messages` (0-16 mensajes previos como datos, no como instrucciones). En imagen: `provider` y `model` del catálogo, `reference_vars` (fotos del cliente, van primero), `reference_media` (fotos base de la galería de la organización, van después), `caption`, `wait_message` y `timeout_seconds` (60-900, default 300). Para varias tomas usa `shots` (2-10) o `repeat` (2-10), excluyentes entre sí.
- **Ojo:** la clave del proveedor vive en Ajustes → Integraciones, nunca en el nodo; el texto usa la clave de IA de la organización (Ajustes → IA). Hay un tope diario de generaciones por proveedor y cada toma cuenta una. Al compartir o importar un flujo, `reference_media` se quita (son fotos de la galería del dueño) y hay que elegirlas de nuevo; `reference_vars` se conserva. Con varias tomas, `var.<result_var>` queda con la primera, `_1`…`_N` con cada URL y `_cantidad` con el total (útil en `condition`). El simulador usa la IA real en texto; en imagen no llama al proveedor y envía la foto de referencia marcada como ejemplo.

### `external_task` — esperar a que tu servidor genere y entregue archivos

- **Hace:** llama por webhook firmado a TU servidor (URL `webhook_url`) y deja la conversación esperando en el nodo. Tu agente produce los archivos (video, PDF, imagen, audio), los sube por MCP y cierra la tarea; la plataforma los envía a la persona y el flujo sigue por `ok`. La conversación no sale del flujo, así que los pagos se siguen validando solos.
- **Úsalo para:** entregar algo que genera un servidor propio: un video personalizado tras el pago, o una imagen con el nombre de la persona antes de pagar. No sirve para generar con los proveedores de la plataforma (usa `ai_generate`).
- **Salidas:** `ok`, `error` y `timeout`. Conecta siempre `error` y `timeout`: sin conectar, la conversación pasa a revisión del equipo con una nota del motivo y queda marcada como entrega fallida.
- **Ajustes clave:** `webhook_url` (https), `wait_message` (aviso mientras espera, admite `{{var.*}}`), `timeout_minutes` (1-60, por defecto 10), `busy_notice` y `busy_message` (qué responder si la persona escribe mientras espera).
- **Ojo:** el secreto del webhook se genera en Ajustes → Integraciones (solo owner/admin, se muestra una vez). Los archivos subidos viven 3 días. Al compartir un flujo se quita la URL del webhook. Cómo debe responder el agente: skill `ares-tarea-externa`.

### `condition` — bifurcación sin IA

- **Hace:** evalúa reglas en orden; la primera que se cumple gana y sale por la salida con el id de esa regla. Si ninguna se cumple, sale por `fallback` (obligatorio conectarlo).
- **Úsalo para:** decidir con datos sin gastar IA. Lee, no modifica (`tag_action` es el que escribe).
- **Reglas posibles:** por etiqueta del contacto, por etapa de la conversación, o por variables: monto o moneda de un pago (`pago.ultimo.monto`, `pago.ultimo.moneda`), cantidad de pagos (`pago.cantidad`), total validado por moneda, nombre del contacto. Con operadores de texto y numéricos.
- **Ojo:** si usas variables de un `wait_payment` concreto, indica cuál con `source_payment_key`. Una variable sin dato hace falsa su condición.

### `tag_action` — poner o quitar una etiqueta

- **Hace:** agrega o quita una etiqueta del contacto y sigue por `next`. Sin IA ni pausa.
- **Úsalo para:** segmentar y, más adelante, decidir con `condition`.
- **Ojo:** la etiqueta debe existir en la organización. Sin etiqueta elegida no hace nada, y no avisa.

### `capi_event` — avisar a Meta que pasó algo valioso

- **Hace:** reporta un evento de mensajería (`LeadSubmitted`, `QualifiedLead`, `InitiateCheckout`, `ViewContent`, `AddToCart`) para quien llegó por un anuncio, y sigue por `next`. Una vez por persona, flujo y evento.
- **Úsalo para:** marcar el momento valioso (interesada, calificada, a punto de pagar). `Purchase` no se elige aquí: lo envía la plataforma al validar el pago.
- **Ojo:** colócalo DESPUÉS de una respuesta de la persona; al inicio contaría cada conversación como lead. Solo sale si hay clic de anuncio y un dataset ligado; en el simulador nunca envía.

### `wait_payment` — esperar y validar un comprobante

- **Hace:** deja la conversación «esperando pago»; cuando llega un comprobante, lo lee (OCR) y lo clasifica.
- **Salidas:** `validated` (pago válido), `rejected` (no se pudo leer o fecha vencida; no implica fraude), `pending_manual_review` (monto alto o posible duplicado) y, si defines precios esperados, `amount_mismatch` («Monto distinto»). Todas deben estar conectadas.
- **Precios esperados (`expected_prices`):** una fila por moneda y sin conversión. Un monto menor, otra moneda o un comprobante raro va a revisión manual con la razón anotada; `accept_higher` permite que un monto mayor también valide.
- **Patrones:** `pending_manual_review` y a veces `rejected` van a un `message` y luego a `handoff_agent`. Para dejar reintentar tras `amount_mismatch`, conéctala de vuelta a un `wait_payment`. Varios `wait_payment` en un flujo permiten cobrar varios pagos (producto más venta adicional).
- **Después de `validated`:** lo que cuelga de esa salida es la entrega del producto; con `is_delivery` en un `message`, se entrega aunque la persona haya salido del flujo.

### `remarketing_template` — reabrir con una plantilla de WhatsApp

- **Hace:** manda una plantilla aprobada por Meta con exactamente dos botones de respuesta rápida, fuera de la ventana de 24 horas, y rutea la respuesta por `yes` o `no`.
- **Salidas:** `yes`, `no` y `unclear`; esta última debe llegar a un `handoff_agent` (directo o por más pasos).
- **Ojo:** es el más estricto al validar: plantilla inexistente, no aprobada o sin exactamente dos botones bloquea publicar.

### `handoff_agent` — pasar al equipo

- **Hace:** entrega la conversación a una persona del equipo, con una nota interna opcional; etapa `revision_manual` (por defecto) o `revision_agente`.
- **Úsalo para:** cerrar cualquier salida de «no puedo resolver esto».
- **Salidas:** ninguna: es terminal.

## Cómo armarlo por MCP

1. Parte de un flujo que ya exista. Por MCP no se crea un flujo vacío ni se cambia su nodo inicial. Si hay un flujo parecido (`list_flows`), **duplícalo con `duplicate_flow`** y adáptalo: es la vía por defecto, no armes desde cero lo que ya está hecho. La copia nace inactiva y hereda el producto. Solo si no hay ningún flujo del que partir, pide a la persona que lo cree en el dashboard (por ejemplo desde la Galería de flujos) y trabaja sobre él.
2. Planifica la lista completa de nodos y conexiones antes de escribir, y créalos todos; no dejes el flujo a medias.
3. `edit_flow` con `graphOps`. Las operaciones son:
   - `{ "op": "add_node", "tempId": "n1", "type": "message", "config": {…}, "positionX": 0, "positionY": 0 }`
   - `{ "op": "add_edge", "tempId": "e1", "fromNodeId": "n1", "fromHandle": "next", "toNodeId": "n2" }` (opcional `delaySeconds`)
   - `{ "op": "patch_node", "nodeId": "…", "config": {…} }`, `{ "op": "delete_node", "nodeId": "…" }`, `{ "op": "delete_edge", "edgeId": "…" }`, `{ "op": "patch_edge", "edgeId": "…", "delaySeconds": 5 }`, `{ "op": "move_node", "nodeId": "…", "positionX": 0, "positionY": 0 }`
   - Un `add_edge` del mismo lote puede usar el `tempId` de un `add_node` anterior. Si dudas de la forma, manda `graphOps: []` primero y revisa con `get_flow`.
4. En una cadena lineal, las posiciones pueden ser todas 0; en ramas, separa los nodos verticalmente para que se lean en el editor.
5. Escribe cada texto con las palabras exactas de la persona; nunca dejes marcadores como «TODO» o «texto aquí».
6. `get_flow` y cuenta los nodos contra tu lista. Luego `validate_flow`, `simulate_flow`, `diff_flow` y `publish_flow` (ver `ares-flujos`).

## Recetas

**Venta con comprobante.** `message` (presentación) → `choice` o `question` (¿quieres comprar?) → `message` (datos de pago, con botones de copiar) → `wait_payment`. `validated` → `message` de entrega (con `is_delivery`) · `rejected` → `message` pidiendo otro comprobante → de vuelta a `wait_payment` · `pending_manual_review` → `message` → `handoff_agent`. Con precios, conecta también `amount_mismatch`.

**Calificar con IA.** `message` → `ai_checkpoint` con `objective` y salidas «interesado» y «no interesado». «Interesado» → `capi_event` (`QualifiedLead`) → `message` de oferta. `timeout`, `max_turns` y `support` → `message` → `handoff_agent`.

**Premiar por monto.** `wait_payment` → `validated` → `condition` con regla de variables sobre `pago.ultimo.monto` → `message` distinto por rama, y `fallback` al mensaje estándar.

**Pedir datos y usarlos.** `message` → `capture` (nombre, teléfono, ciudad) → `next` a un `message` con `{{var.nombre}}` · `invalid` → `message` → `handoff_agent`. Con `condition` sobre `var.ciudad` puedes separar por zona.

**Mostrar planes con imagen.** `carousel` con una tarjeta por plan y un botón `reply` por tarjeta → cada botón a su `message` o a `wait_payment`; `unclear` a un `message` que pida pulsar un botón y de vuelta al carrusel o a `handoff_agent`.

**Reabrir una conversación fría.** `remarketing_template` → `yes` a `message` o `ai_checkpoint`; `no` a `tag_action` (etiqueta «no interesado»); `unclear` a `handoff_agent`.

## Errores comunes

- Dejar sin conectar `timeout` del agente de IA, `unclear` de `choice` o `carousel`, `invalid` de `capture` (con campos obligatorios) o `fallback` de `condition`: bloquea publicar.
- Poner `capi_event` al inicio del flujo.
- Armar un flujo desde cero cuando existe uno parecido: duplícalo con `duplicate_flow`.
- Escribir cuentas, CLABE o números de pago como texto plano sin que la persona lo haya pedido: usa `copy_buttons`.
- Mezclar espera sin límite con tiempos límite o salidas de «sin respuesta».
- Usar `question` para más de dos opciones.
- Usar `{{var.x}}` o una condición de variables sin un `capture` antes que guarde ese dato.
- Describir las ramas en el texto de la IA: las ramas se dibujan con salidas.

Para el nodo «Agente de IA» y las utilidades, usa la skill ares-nodos-ia.
