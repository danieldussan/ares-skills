---
name: ares-estado
description: Usa esta skill cuando el usuario pregunte cómo va su operación en el bot de WhatsApp en este momento — conversaciones por revisar o sin responder, pagos pendientes, ventas de hoy o si sus bots están conectados y con buena calidad. Es solo lectura, a través del MCP.
---

# Estado de la operación por MCP

Todas las tools de esta skill son de lectura: sirven a cualquier rol y no cambian nada. Para editar flujos usa la skill `ares-flujos`; para análisis de un periodo, `ares-reportes`.

## Qué tool usar

| La persona pregunta por…                        | Tool                           |
| ----------------------------------------------- | ------------------------------ |
| Ventas de hoy                                   | `get_daily_stats`              |
| Pagos por revisar, validados o rechazados       | `get_payments`                 |
| Chats que esperan respuesta o están en revisión | `get_conversations`            |
| Si los bots están conectados y su calidad       | `get_whatsapp_instance_status` |
| Qué flujos hay y cuáles están activos           | `list_flows`                   |

## Conversaciones (`get_conversations`)

- Filtra por `state` con la etapa exacta: `nuevo`, `en_flujo`, `flujo_terminado`, `revision_manual`, `revision_agente`, `en_ai`, `esperando_pago`, `pago_confirmado`, `en_remarketing`.
- «¿Qué requiere atención del equipo?»: `state` = `revision_manual` o `revision_agente` (una llamada por etapa), más `unanswered: true` para los que nadie ha contestado y `hasUnread: true` para los no leídos.
- «¿A quién no se le entregó?»: `deliveryIssue: "any"` trae solo las conversaciones con una entrega fallida (o un código: `external_task_error`, `external_task_timeout`, `payment_delivery_failed`, `ai_batch_undelivered`). Cada item trae `deliveryIssue` (`{ code, at, detail }` o `null`). Es una marca aparte de la etapa: la conversación puede seguir en `en_flujo` o `esperando_pago`. Cuando ya se entregó a mano (o se decidió no entregar), `clear_delivery_issue { conversationId }` (propietario o administrador) quita la marca; no envía nada ni cambia la etapa.
- Otros filtros: `flowId`, `fromAd` (llegaron por anuncio), `adName`, `search`, `instanceId` (el bot).
- Pagina por cursor: la respuesta trae `hasMore`; para la siguiente página pasa `cursorUpdatedAt` y `cursorId` del último item. No hay total exacto: no lo inventes, di «hay más» o cuenta lo que trajiste.
- `pageSize` por defecto 20: sube el valor si necesitas contar, pero no pidas todo el historial.

## Pagos (`get_payments`)

- `state`: `pending_manual_review` (esperan a una persona), `validated` (venta), `rejected`.
- Filtros: `flowId`, `instanceId`, `from` y `to` (fecha `AAAA-MM-DD`), `phone`, `page` y `pageSize` (máximo 100), `sortBy`, `sortDir`.
- Solo una venta cuenta como ingreso si está `validated`. Un pago sin `amount` no se puede sumar: dilo, no lo cuentes como cero.
- Los montos vienen en la moneda de cada pago; no los sumes entre monedas distintas. Para totales convertidos usa `get_daily_stats` o `get_reports_kpis`.

## Ventas de hoy (`get_daily_stats`)

Devuelve las ventas validadas de hoy en la zona horaria de la organización. Si `fxMissing` nombra una moneda, el total y el promedio salen `null` por falta de tasa: repórtalo así, nunca como 0.

## Bots (`get_whatsapp_instance_status`)

Por cada bot: nombre (`label`), si está activo, su flujo, la calidad (`quality_rating`), el límite de mensajería (`messaging_limit_tier`), cuándo se revisó la calidad y su número. No trae tokens ni secretos.

- Calidad baja o límite bajo: avisa a la persona; puede frenar envíos.
- `quality_checked_at` viejo o vacío: la calidad puede estar desactualizada, dilo.
- Si algo parece caído, esta tool no lo arregla: indica que lo revise en el dashboard (menú Bots).

## Cómo responder

- Empieza por la respuesta («Tienes 3 pagos por revisar») y luego el detalle en una tabla corta.
- Un error o dato desconocido se dice tal cual: «no pude leerlo» o «—», nunca como cero o vacío.
- En el texto para la persona habla siempre de «bots».
- Nunca muestres identificadores internos ni identificadores de WhatsApp: presenta al contacto por su nombre o teléfono.
