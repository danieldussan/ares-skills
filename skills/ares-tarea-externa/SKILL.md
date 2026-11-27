---
name: ares-tarea-externa
description: Usa esta skill cuando tu servidor o agente reciba un webhook «external_task.created» de la plataforma y deba producir archivos (video, PDF, imagen, audio) para una conversación. Explica cómo verificar la firma, cómo subir los archivos y cómo cerrar la tarea por MCP, con topes y casos de error.
---

# Tarea externa: lo que debes hacer cuando llega el webhook

El nodo «Tarea externa» de un flujo llama a tu servidor y deja la conversación esperando. Tú produces lo pedido (por ejemplo un video), lo subes y cierras la tarea por MCP. La plataforma envía lo subido a la persona y el flujo sigue.

## 1. Qué llega

`POST` a la URL del nodo, `Content-Type: application/json`, con las cabeceras `X-Ares-Task-Id` y `X-Ares-Signature: t=<unix>,v1=<hex>`.

Cuerpo: `event` (`external_task.created`), `task_id`, `created_at`, `expires_at`, `flow {id,name}`, `node_id`, `conversation_id`, `bot_id`, `contact {phone}`, `vars` (variables del flujo) y `payment {id,amount,currency,validated_at}` o `null`.

Responde **2xx de inmediato** y trabaja después. Cualquier otra respuesta (o un error de red tras un reintento) marca la tarea como fallida y el flujo sale por «error». Si ya recibiste ese `task_id`, ignóralo.

## 2. Verifica la firma

`v1 = HMAC_SHA256(secreto, "<t>.<cuerpo crudo>")` en hex. Usa el cuerpo **crudo**, sin volver a serializarlo. Rechaza si `t` difiere más de 300 s de tu reloj. El secreto se genera en Ajustes → Integraciones (se muestra una sola vez).

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
export function verify(raw: string, header: string, secret: string): boolean {
  const [t, v1] = header.split(",").map((p) => p.split("=")[1] ?? "");
  const want = createHmac("sha256", secret).update(`${t}.${raw}`).digest("hex");
  const ok = v1.length === want.length && timingSafeEqual(Buffer.from(v1), Buffer.from(want));
  return ok && Math.abs(Date.now() / 1000 - Number(t)) < 300;
}
```

```python
import hmac, hashlib, time
def verify(raw: bytes, header: str, secret: str) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t, v1 = parts.get("t", ""), parts.get("v1", "")
    want = hmac.new(secret.encode(), t.encode() + b"." + raw, hashlib.sha256).hexdigest()
    return hmac.compare_digest(v1, want) and abs(time.time() - int(t)) < 300
```

## 3. Orden de llamadas (tools MCP, rol `agent`)

1. Por **cada archivo**: `request_task_upload` con `{ taskId, filename, mimeType, sizeBytes }`. Devuelve `{ key, uploadUrl, headers, expiresInSeconds }`.
2. `PUT` de los bytes a `uploadUrl` con el **mismo** `Content-Type` que declaraste. La URL dura 10 minutos.
3. Una sola vez: `complete_external_task` con `{ taskId, outcome, messages?, vars?, errorNote? }`.

`outcome` es `ok` o `error`. Cada elemento de `messages` es `{ type: "text", text }` o `{ type: "image" | "video" | "audio" | "document", key, caption?, filename? }`, donde `key` es la recibida al pedir la subida. `messages` solo se acepta con `ok`.

Ejemplo de cierre correcto: sube un mp4, luego `complete_external_task` con `outcome: "ok"`, un mensaje de video con su `key` y `caption`, y `vars` si el flujo debe usar un dato nuevo.

## 4. Topes

- Hasta 10 archivos por tarea y 10 mensajes por cierre. Texto ≤ 4096, `caption` ≤ 1024, `filename` ≤ 200.
- Hasta 20 `vars`: nombre `^[a-z][a-z0-9_]{0,39}$`, valor ≤ 1000. `errorNote` ≤ 500.
- Imagen jpeg, png o webp: 10 MB. Video mp4 (no QuickTime): 50 MB. Audio mpeg, ogg o mp4: 16 MB. Documento pdf: 50 MB.
- En un bot de Meta Cloud el video no puede pasar de 16 MB.
- El tipo y el contenido real se comprueban: un archivo con firma distinta a su `mimeType` se rechaza.

Si un archivo falla en `complete_external_task`, la respuesta indica el índice del mensaje; se borra solo ese archivo y la tarea sigue pendiente: súbelo de nuevo y cierra otra vez.

## 5. Vencimiento y errores

- La tarea tiene un plazo (`expires_at`, por defecto 10 minutos). Pasado el plazo la plataforma sigue por «timeout» y un cierre tardío devuelve «La tarea venció».
- Si la conversación ya avanzó (la tomó una persona, cambió de flujo o se reinició), `complete_external_task` responde `discarded`: nada se entregará. Si aún quieres escribirle a la persona, usa `send_message` en revisión y luego `clear_delivery_issue`.
- Si no puedes producir el resultado, cierra con `outcome: "error"` y una `errorNote` breve y clara: el flujo sale por «error» y la nota queda para el equipo.
- `complete_external_task` es **idempotente**: el primer cierre gana y repetirlo devuelve `already_completed` sin efecto; `discarded` significa que la conversación ya no espera la tarea. Si responde «Inténtalo de nuevo», repite la misma llamada sin miedo.

Si la tarea vence o cierras con error, la conversación queda marcada (`deliveryIssue` = `external_task_timeout` o `external_task_error`). Para encontrar a quién no se le entregó, usa `get_conversations` con `deliveryIssue: "any"`. Cuando ya se lo entregaste con `send_message` (o decides no hacerlo), llama a `clear_delivery_issue { conversationId }` (propietario o administrador) para quitar la marca y que la lista siga siendo una lista de pendientes.

## 6. Archivos

Los archivos subidos viven 3 días y solo sirven para esa tarea. No uses la galería de medios: no hace falta y no se mezcla con este flujo. Nunca guardes ni registres las URLs firmadas.
