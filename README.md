# Ares Skills

Skills de agente para trabajar con la plataforma Ares (bots de WhatsApp) por MCP: crear
y editar flujos, armar nodos, diagnosticar bots, consultar el estado de la operación y
leer reportes. Siguen el estándar abierto [Agent Skills](https://agentskills.io/): una
carpeta por skill con su `SKILL.md`.

> `skills/` y `VERSION` son generados automáticamente desde `whatsapp-bot`. No los edites
> aquí: cualquier cambio se sobrescribe en la próxima sincronización.

## Instalación (cualquier agente)

Requiere Node.js 22.20 o superior. El instalador oficial es el CLI de Vercel:

```bash
# Ver qué hay antes de instalar
npx skills add danieldussan/ares-skills --list

# Instalar las 9 skills para tu usuario, en los agentes detectados
npx skills add danieldussan/ares-skills -g --skill '*' -y

# Elegir agentes concretos
npx skills add danieldussan/ares-skills -g -a claude-code -a opencode -a codex -a cursor

# Fijar una versión (recomendado en equipos)
npx skills add danieldussan/ares-skills@v1.0.1 -g --skill '*' -y
```

Actualizar:

```bash
npx skills update -g
npx skills list -g
```

El instalador escribe en `.agents/skills/` y crea los enlaces para cada agente
(`.claude/skills/`, `~/.config/opencode/skills/`, `~/.agents/skills/`, etc.).

### Claude Code (alternativa con marketplace)

```text
/plugin marketplace add danieldussan/ares-skills
/plugin install ares-skills@ares-skills
```

Las skills del plugin se invocan con prefijo, por ejemplo `/ares-skills:ares-bots`.

## Requisito: MCP de Ares

Estas skills documentan tools reales del servidor MCP de la plataforma. Para que
funcionen, configura el MCP de Ares en tu agente. Ninguna skill pide ni muestra tokens:
las credenciales se gestionan en el dashboard de la plataforma.

## Skills incluidas

| Skill | Para qué sirve |
| --- | --- |
| `ares-flujos` | Crear, editar o revisar flujos por MCP y escribir las instrucciones de la IA. |
| `ares-nodos` | Saber qué hace cada tipo de nodo, cuándo usarlo y cómo armar flujos con recetas. |
| `ares-nodos-ia` | Armar, mejorar o revisar el nodo «Agente de IA» y las utilidades del flujo. |
| `ares-bots` | Conectar, revisar y diagnosticar bots de WhatsApp: estado, calidad, límite y caídas. |
| `ares-estado` | Ver el estado de la operación: conversaciones por revisar, pagos y ventas de hoy. |
| `ares-reportes` | Reportes de ventas, conversión, gasto de anuncios y ROAS, con comparación de periodos. |
| `ares-revision-ia` | Revisar cómo responde la IA de un flujo con conversaciones reales. |
| `ares-tarea-externa` | Atender webhooks `external_task.created`, subir archivos y cerrar la tarea. |
| `ares-atencion` | Responder, tomar o cerrar conversaciones en revisión manual o del agente. |

## Versiones

El número de versión vive en `VERSION` y se publica como tag `vX.Y.Z` con un ZIP por
skill en la pestaña Releases. Para distribuir una versión fija, usa la forma
`npx skills add danieldussan/ares-skills@vX.Y.Z`.
