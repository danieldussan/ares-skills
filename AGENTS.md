# Instrucciones para agentes en este repo

Este repositorio distribuye las skills oficiales de Ares.

- `skills/` y `VERSION` son **generados** desde `backend/agent-skills/` del repo
  `whatsapp-bot`. No los edites ni los muevas: un workflow los sobrescribe.
- Si encuentras un error en una skill, corrígelo en `whatsapp-bot` (rama correspondiente),
  no aquí.
- `README.md`, `llms.txt` y los archivos de `.claude-plugin/` sí se mantienen a mano en
  este repo.
- Las skills nombran tools del MCP de Ares. Antes de cambiar una skill, confirma que la
  tool existe en el servidor; el test `backend/src/mcp/agentSkill.test.ts` lo verifica.
- Todo el texto en español usa tuteo neutral, nunca voseo. Tampoco se usa la palabra
  «instancia» para referirse a los bots: se dice «bot».
- Ninguna skill debe pedir, mostrar ni guardar tokens ni credenciales.
