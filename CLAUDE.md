# CLAUDE.md

## Biblioteca de agentes especializados — consultar sempre

Antes de iniciar qualquer tarefa de desenvolvimento neste projeto (implementação, revisão de arquitetura, code review, design de API, planejamento de tarefas, etc.), **sempre verificar a pasta**:

`C:\Users\lucas\Documents\Desenvolvimento\Agentes`

Essa pasta contém uma biblioteca de agentes especialistas (`*.agent.md`) com contexto sênior (Engenharia de Software, Arquitetura, Segurança, Performance, Testes, Observabilidade) já validado pelo usuário. Ler o `README.md` da pasta para o índice completo e critérios de seleção antes de escolher um agente.

Agentes mais relevantes para este projeto (proxy compatível com APIs de IA, multi-backend via CLIs):

- **ai-architecture-reviewer** — revisão de arquitetura de sistemas de IA, agentes, prompts e pipelines Node.js; conhece Claude, GPT, Gemini, DeepSeek, Qwen, MCP, Claude Code. Usar para validar decisões estruturais do gateway e do sistema de adapters.
- **nodejs-specialist** — Node.js, Express, middleware, APIs REST. Usar para implementação do `packages/server`.
- **code-reviewer** / **code-quality-expert** — usar após qualquer entrega de código antes de considerar a tarefa concluída.
- **task-specialist** / **product-manager** — usar para quebrar epics em tarefas e priorizar escopo (ex.: fases de expansão para novos backends CLI).

Regra prática: se a tarefa é sobre construir/revisar este produto, escolher o agente de domínio adequado na pasta acima antes de agir; não reinventar o papel de um agente já definido lá.
