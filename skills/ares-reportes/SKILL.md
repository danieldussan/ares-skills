---
name: ares-reportes
description: Usa esta skill cuando el usuario pida un reporte o análisis de ventas, ingresos, conversión, gasto de anuncios o ROAS de un periodo, o quiera comparar periodos y explicar qué cambió. Solo lectura, a través del MCP.
---

# Reportes por MCP

Solo lectura y abierta a cualquier rol. Para el estado de este momento usa `ares-estado`.

## Tools

- `get_reports_kpis` con `from` y `to` (`AAAA-MM-DD`, en la zona horaria de la organización). Sin fechas toma los últimos 30 días. Son los mismos números que la página Reportes.
- `get_daily_stats`: solo las ventas de hoy.
- `get_payments` con `state: "validated"` y rango de fechas: para ver las ventas una por una (por ejemplo, por flujo con `flowId`).
- `list_flows`: para nombrar los flujos que aparezcan en los datos.

## Qué significa cada número de `get_reports_kpis`

| Campo                                     | Significado                                                                                         |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `revenueTotal`, `salesCount`, `avgTicket` | Ventas validadas del rango (por fecha de validación), convertidas a la moneda local `currency`      |
| `conversationsCount`                      | Conversaciones NUEVAS del rango, contadas por contacto y flujo                                      |
| `cohortBuyers`, `conversionRate`          | Compradores de esas conversaciones nuevas, y su porcentaje sobre ellas (solo pagos del mismo flujo) |
| `pendingCount`                            | Pagos que esperan revisión                                                                          |
| `adSpendTotal`                            | Gasto de anuncios atribuido en el rango                                                             |
| `roas`                                    | Todas las ventas ÷ gasto atribuido (incluye ventas sin anuncio)                                     |
| `roasAds` y `adRevenueTotal`              | Solo ventas con clic de anuncio del mismo flujo en los 30 días previos al pago ÷ el mismo gasto     |

`roasDefinition` y `roasAdsDefinition` traen la definición oficial: cítala cuando expliques un ROAS. Nunca calcules el ROAS por tu cuenta: usa el que devuelve la tool.

## Reglas para no mentir con los datos

- `null` es «no se pudo calcular», no cero. Si `fxMissing` nombra una moneda sin tasa, el ingreso o el gasto salen `null`: dilo y di qué moneda falta.
- Lee las advertencias (`revenueWarning`, `spendWarning`, `amountMissingWarning`) y `amountMissingCount` (ventas sin monto): menciónalas junto al número afectado.
- `fxApproximate` indica tasas aproximadas: avísalo cuando sea `true`.
- Si la respuesta trae `error` (por ejemplo, rango inválido), repórtalo y corrige el rango; no inventes cifras.
- No sumes montos de monedas distintas a mano.
- Los pagos simulados nunca cuentan: las tools ya los excluyen.

## Cómo comparar periodos

1. Llama `get_reports_kpis` una vez por periodo, con rangos de la misma duración.
2. Compara `revenueTotal`, `salesCount`, `avgTicket`, `conversionRate`, `adSpendTotal` y `roas`, con el cambio absoluto y en porcentaje; si el periodo base es 0 o `null`, escribe «—».
3. Para explicar un cambio, baja a `get_payments` por flujo y mira qué flujo movió el total.
4. Distingue lo que muestran los datos de lo que supones: sin evidencia, di «no se puede saber con estos datos».

## Formato de la respuesta

Resumen de una línea, tabla corta con los KPIs, y al final las advertencias de datos. Cifras con la moneda `currency`. En el texto para la persona usa siempre «bots».
