---
name: ares-bots
description: Usa esta skill cuando el usuario quiera conectar, revisar o diagnosticar los bots de WhatsApp de su cuenta, por ejemplo cómo conectar un bot, si está conectado, su calidad o límite, por qué no envía, bloqueos, caídas de línea o qué puede hacerse por MCP y qué solo en el dashboard.
---

# Bots de WhatsApp

Un **bot** es una línea de WhatsApp conectada a la cuenta, con un flujo asignado. Tiene dos canales posibles: **QR** (se vincula con el celular) o **Meta Cloud** (API oficial de WhatsApp). En el texto para la persona habla siempre de «bots», «QR» y «Meta Cloud».

## Qué puedes hacer por MCP y qué no

El MCP solo permite LEER los bots. Conectar, desconectar, crear, borrar o cambiar un bot se hace en el dashboard, y es parte de tu trabajo decirle a la persona dónde y cómo.

| Necesidad                                         | ¿Por MCP? | Tool o lugar                                  |
| ------------------------------------------------- | --------- | --------------------------------------------- |
| Ver bots, su estado, calidad y límite             | Sí        | `get_whatsapp_instance_status`                |
| Ver qué flujo usa cada bot                        | Sí        | `get_whatsapp_instance_status` y `list_flows` |
| Conversaciones de un bot                          | Sí        | `get_conversations` con `instanceId`          |
| Pagos de un bot                                   | Sí        | `get_payments` con `instanceId`               |
| Cambiar el flujo de un bot, activar o desactivar  | No        | Dashboard, menú Bots                          |
| Crear un bot y conectarlo (QR o Meta Cloud)       | No        | Dashboard, menú Bots                          |
| Desconectar, cambiar de número, marcar un bloqueo | No        | Dashboard, detalle del bot                    |

Las tools de escritura del MCP editan flujos, no bots. Para diagnóstico de resultados usa `ares-estado`; para reportes, `ares-reportes`.

## Revisar el estado de los bots

Llama a `get_whatsapp_instance_status`. Por cada bot trae: nombre (`label`), si está activo (`is_active`), su flujo (`flow_id`), la calidad (`quality_rating`), el límite de mensajería (`messaging_limit_tier`), cuándo se revisó (`quality_checked_at`) y su número (`display_phone_number`). Nunca trae tokens ni secretos.

Cómo leerlo:

- **Inactivo** (`is_active` falso): no está atendiendo. Puede estar sin conectar, desactivado por la persona o desactivado por exceso de plan.
- **Sin flujo** (`flow_id` vacío): el bot recibe pero no hay recorrido que lo atienda; la persona debe asignarle uno (un flujo archivado no se puede asignar).
- **Calidad** (`quality_rating`): si baja, WhatsApp puede limitar los envíos. Avisa a la persona antes de lanzar campañas.
- **Límite** (`messaging_limit_tier`): techo de conversaciones nuevas (de 250 a ilimitado). Cuanto más bajo, menos campañas masivas caben.
- **`quality_checked_at` viejo o vacío:** se refresca cada 2 horas; si no cambia, dilo en vez de afirmar la calidad.

Para ver si un bot trabaja, cruza con datos: `get_conversations` con su `instanceId` y `get_payments` con su `instanceId`. Un bot activo sin conversaciones recientes puede estar desconectado aunque figure activo.

## Cómo se conecta un bot (guía para la persona)

1. En el dashboard, menú **Bots**: crear un bot nuevo (solo propietario o administrador).
2. Elegir el canal:
   - **QR:** el sistema genera un código; en el celular, WhatsApp → Dispositivos vinculados → Vincular un dispositivo, y escanearlo. Si la persona indica el número, puede usar un código de emparejamiento en lugar del QR. El bot aparecerá en el celular con el nombre «Ares».
   - **Meta Cloud:** requiere los datos de la cuenta de WhatsApp Business en Meta (token, identificador del número y de la cuenta). La guía de permisos está en Ajustes.
3. Esperar a que el estado pase de «conectando» a conectado. No repetir la conexión mientras esté en curso.
4. Asignarle un flujo y activarlo.

Reglas que conviene advertir:

- **Un número solo puede estar activo en UN bot de toda la plataforma.** Si ya está activo en otro, la conexión se rechaza.
- **Límite del plan:** la cantidad de bots conectados depende de la suscripción. Si la cuenta tiene un pago vencido sin resolver, no se pueden agregar bots hasta pagar.
- Un bot QR que nunca llegó a conectar se borra solo a los 2 días; los borradores sin canal, a los 7.
- Cambiar el canal de un bot ya conectado (de QR a Meta Cloud o al revés) no existe todavía: solo se adjunta canal a un bot sin canal.

## Si un bot no envía o se cayó

Pregunta o revisa en este orden:

1. **¿Está activo y conectado?** `get_whatsapp_instance_status`. Si no, que la persona lo reconecte desde el dashboard.
2. **¿Tiene el estado «bloqueado», «restringido» o «por verificar»?** El dashboard lo muestra en el detalle del bot.
   - _Restringido_: lo fija el sistema; solo bloquea chats nuevos.
   - _Bloqueado_: un bot bloqueado no envía; lo marca una persona.
   - _Por verificar_: tras una caída por cierre de sesión, el sistema pregunta si WhatsApp bloqueó el número. Responder «Sí, fue un bloqueo» activa el modo recuperación: silencio y luego envíos escalonados.
3. **¿Se cayó la línea?** Un bot QR caído retiene los envíos programados hasta 24 horas y los libera, de a poco, solo si reconecta con el MISMO número. Con otro número, se descartan. Los envíos de pago nunca se descartan.
4. **¿Cuenta suspendida o por exceso de plan?** Un bot marcado por exceso de plan queda inactivo y no se reactiva solo.
5. **¿El flujo está activo y asignado?** Un bot sin flujo no responde.

## Cómo responder

- Empieza por el hallazgo («2 de tus 3 bots están activos; el de Ventas no tiene flujo») y luego una tabla corta: bot, activo, flujo, calidad, límite.
- Un dato desconocido es «—», con su porqué; nunca lo muestres como bueno ni como cero.
- No expliques la integración técnica ni nombres del proveedor del canal QR: di «QR».
- No pidas ni muestres tokens. Si la persona necesita datos de Meta, guíala al dashboard.
