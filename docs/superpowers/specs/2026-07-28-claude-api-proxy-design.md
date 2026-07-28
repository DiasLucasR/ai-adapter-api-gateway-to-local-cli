# Claude API Proxy — Design (MVP)

Data: 2026-07-28

## 1. Objetivo

Gateway local que expõe endpoints compatíveis com as APIs oficiais **Anthropic Messages API** e **OpenAI Chat Completions API**, mas que executa as requisições usando ferramentas de IA já instaladas localmente (CLIs com assinatura/login próprios), começando pelo **Claude Code CLI** via **Claude Agent SDK**.

Objetivo central de flexibilidade: o formato da API que o cliente usa para chamar (Anthropic ou OpenAI) é **independente** de qual backend realmente processa a requisição. Um cliente pode chamar no formato Anthropic pedindo `"model": "claude-sonnet"` e o sistema, por configuração, redirecionar isso para o GitHub Copilot CLI, OpenAI, ou qualquer adapter cadastrado. Essa é uma ferramenta de testes/desenvolvimento local — não um serviço para terceiros.

## 2. Escopo do MVP

**Dentro do escopo:**
- Rota compatível `POST /v1/messages` (Anthropic Messages API), incluindo streaming SSE.
- Rota compatível `POST /v1/chat/completions` (OpenAI Chat Completions API), incluindo streaming SSE.
- Um adapter de backend: `claude-code`, implementado sobre o **Claude Agent SDK**, usando a sessão local já autenticada do Claude Code (sem necessidade de API key paga da Anthropic).
- De-para configurável entre o nome do "model" recebido em qualquer rota e o adapter/target real que processa (`models.yaml`).
- Autenticação local simples (API key própria gerada pelo sistema).
- Rate limiting básico, logs estruturados, contagem de tokens.
- App Electron: liga/desliga o servidor, edita o de-para de modelos, gerencia credenciais, mostra status dos adapters, e exibe a documentação de referência de cada API/serviço suportado.

**Fora do escopo do MVP (fase 2, arquitetura já preparada para isso):**
- Novos adapters de backend (Codex CLI, GitHub Copilot CLI, Gemini CLI, Qwen Code CLI, DeepSeek Coder CLI, OpenRouter). Cada um exige validação individual de viabilidade de auth/streaming antes de ser prometido.
- `tool_use` / function calling ponta a ponta (definição, execução e retorno de tools do cliente).
- Rotas de compat adicionais além de Anthropic/OpenAI (ex. DeepSeek nativo).
- Docker Compose como forma alternativa de execução headless (o server já nasce desacoplado do Electron para permitir isso depois, mas não é entregue no MVP).

## 3. Viabilidade técnica (por que isso funciona)

O Claude Code CLI tem um modo não-interativo (`claude -p`) que usa a mesma autenticação OAuth da sessão paga do app (Pro/Max), não uma API key separada. O **Claude Agent SDK** (oficial, Node/TS e Python) é a camada suportada para automatizar isso — ele invoca o CLI, gerencia o processo e devolve eventos estruturados (mensagens, streaming, uso de tokens), evitando parsing frágil de texto de terminal ou automação de browser.

**Restrição de uso reconhecida:** os Termos de Uso da Anthropic restringem usar a autenticação de uma assinatura pessoal para revender/servir terceiros como se fosse a API paga oficial. Este sistema é declaradamente de uso pessoal e local — um único usuário, na própria máquina, testando ferramentas contra a própria assinatura já paga. Não deve ser exposto publicamente nem usado para servir múltiplos usuários externos.

## 4. Arquitetura

```
Cliente externo (SDK Anthropic, SDK OpenAI, ferramentas de terceiros)
        │  HTTP (Bearer/x-api-key local)
        ▼
┌───────────────────────────────────────────────┐
│  packages/server (Node + TypeScript + Express)  │
│                                                  │
│  Gateway                                        │
│   - Auth (API key local)                        │
│   - Rate limit (in-memory, single user)          │
│   - Logging estruturado (pino)                   │
│   - Validação de payload (zod)                   │
│                                                  │
│  Compat Routes (inbound adapters — ports)        │
│   - POST /v1/messages          (Anthropic)       │
│   - POST /v1/chat/completions  (OpenAI)          │
│      converts request  → UnifiedRequest          │
│      converts response ← UnifiedResponse          │
│      converts stream   ← UnifiedStreamEvent       │
│                                                  │
│  Núcleo (domínio, sem conhecimento de formato    │
│  externo): UnifiedRequest / UnifiedResponse /     │
│  UnifiedStreamEvent                              │
│                                                  │
│  Adapter Registry                                │
│   - lê models.yaml (de-para model → adapter)      │
│                                                  │
│  Backend Adapters (outbound adapters — ports)     │
│   - claude-code/ (MVP, via Claude Agent SDK)      │
│   - [fase 2: codex/, github-copilot/, gemini/...] │
│                                                  │
│  Admin API (local, consumida só pelo Electron)    │
│   - POST /admin/config/reload                     │
│   - GET  /admin/adapters/status                    │
│   - GET  /admin/docs (lista referências das APIs)  │
└───────────────────────────────────────────────┘
        ▲  inicia/derruba processo, chama Admin API
        │
┌───────────────────────────────────────────────┐
│  packages/desktop (Electron)                    │
│   - Start/stop do server (child process)         │
│   - Editor do de-para model → adapter             │
│   - Gerência de credenciais (API key local +      │
│     credenciais de cada backend, quando aplicável)│
│   - Toggle de quais compat routes estão ativas    │
│     (Anthropic / OpenAI)                          │
│   - Status de cada adapter (via Admin API)        │
│   - Viewer de documentação: aba por API/serviço    │
│     disponível (Anthropic Messages API, OpenAI     │
│     Chat Completions, Claude Code CLI), com        │
│     referência offline (markdown empacotado) e      │
│     link para a doc oficial online                │
└───────────────────────────────────────────────┘
```

### 4.1 Núcleo Unified (contratos internos)

```ts
interface UnifiedMessage {
  role: "system" | "user" | "assistant" | "tool";
  content: string; // MVP: texto puro. Blocos ricos (imagem, tool_use) ficam para fase 2.
}

interface UnifiedRequest {
  requestedModel: string;   // valor bruto de "model" recebido na rota de entrada
  system?: string;
  messages: UnifiedMessage[];
  maxTokens: number;
  temperature?: number;
  stream: boolean;
}

interface UnifiedResponse {
  text: string;
  stopReason: "end_turn" | "max_tokens" | "stop_sequence" | "error";
  usage: { inputTokens: number; outputTokens: number };
}

type UnifiedStreamEvent =
  | { type: "start" }
  | { type: "text_delta"; text: string }
  | { type: "end"; stopReason: UnifiedResponse["stopReason"]; usage: UnifiedResponse["usage"] };
```

### 4.2 Contrato do Adapter (porta de saída)

```ts
interface ModelAdapter {
  id: string; // "claude-code"
  send(req: UnifiedRequest, targetModel: string): Promise<UnifiedResponse>;
  stream(req: UnifiedRequest, targetModel: string): AsyncIterable<UnifiedStreamEvent>;
  healthCheck(): Promise<{ available: boolean; reason?: string }>;
}
```

O adapter `claude-code` implementa isso usando o Claude Agent SDK: monta o prompt (`system` + histórico de `messages`) e o `target_model` (ex. `sonnet`, `opus`) a partir do de-para, invoca o SDK, e traduz os eventos do SDK para `UnifiedStreamEvent`.

### 4.3 De-para de modelos (`models.yaml`)

```yaml
models:
  claude-sonnet:
    adapter: claude-code
    target_model: sonnet
  gpt-4:
    adapter: claude-code
    target_model: opus
  claude-code/sonnet:
    adapter: claude-code
    target_model: sonnet
```

Qualquer chave em `models.models` pode apontar para qualquer adapter — o nome do model recebido é só uma chave de lookup, sem relação obrigatória com o backend real. O Electron edita esse arquivo através da Admin API (`/admin/config/reload`), que recarrega o registry em memória sem precisar reiniciar o processo.

## 5. Fluxo de request

**Não-streaming:**
1. Cliente → `POST /v1/messages` ou `POST /v1/chat/completions`.
2. Gateway: auth → rate limit → validação de schema (zod, específico de cada formato).
3. Rota converte payload → `UnifiedRequest`.
4. Adapter Registry resolve `requestedModel` em `models.yaml` → adapter + target_model.
5. Adapter `.send()` → `UnifiedResponse`.
6. Rota converte `UnifiedResponse` → JSON no formato exato da API de origem (Messages API ou Chat Completions).
7. Gateway loga (model pedido, adapter usado, tokens, latência, request-id) e devolve.

**Streaming:** idêntico, mas a rota consome `adapter.stream()` e traduz cada `UnifiedStreamEvent` para o dialeto SSE correto:
- Anthropic: `message_start` → `content_block_start` → `content_block_delta` (`text_delta`) → `content_block_stop` → `message_delta` → `message_stop`.
- OpenAI: `chat.completion.chunk` com `choices[0].delta.content`, finalizando com `finish_reason` e o terminador `data: [DONE]`.

O adapter nunca sabe qual dialeto de streaming originou a chamada.

## 6. Mapeamento de erros

Erros internos (falha ao invocar o Claude Agent SDK, model não encontrado no de-para, timeout) são convertidos para o formato de erro nativo de cada rota:

- Anthropic: `{ "type": "error", "error": { "type": "invalid_request_error" | "not_found_error" | "internal_server_error" | ..., "message": "..." } }` com o HTTP status correspondente (400/404/500/529).
- OpenAI: `{ "error": { "message": "...", "type": "invalid_request_error" | "api_error" | ..., "param": null, "code": null } }`.

## 7. Segurança

- API key local própria (gerada na primeira execução do Electron, armazenada fora do repo).
- Toda rota pública exige `x-api-key` (Anthropic) ou `Authorization: Bearer` (OpenAI) validado contra essa key local — não contra a Anthropic/OpenAI reais.
- A Admin API só aceita conexões de loopback (`127.0.0.1`) e nunca fica exposta na mesma porta pública sem autenticação própria.
- Rate limiting simples em memória (o uso é de um único usuário local).
- Nenhuma credencial de backend (ex. sessão do Claude Code) é logada ou exposta nas respostas.

## 8. Observabilidade

- Logs estruturados (pino): request-id, rota, model solicitado, adapter usado, tokens (input/output), latência, status.
- Sem dashboard dedicado no MVP — consulta via arquivo de log; o Electron não exibe stream de logs ao vivo nesta fase.

## 9. Documentação embutida no Electron

Uma aba "Docs" lista as APIs/serviços disponíveis (Anthropic Messages API, OpenAI Chat Completions API, Claude Code CLI) com:
- Referência offline resumida (markdown empacotado no app, gerado a partir do levantamento feito nesta fase de design — ver seção 10).
- Link para abrir a documentação oficial online no navegador padrão do sistema.

## 10. Referência das APIs (levantada via crawler nesta fase)

### Anthropic Messages API (`POST /v1/messages`)
- Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `Content-Type: application/json`.
- Request mínimo: `{ model, max_tokens, messages: [{role, content}], system?, stream? }`.
- Response: `{ id, type: "message", role: "assistant", model, content: [{type:"text", text}], stop_reason, usage: {input_tokens, output_tokens} }`.
- Streaming SSE: `message_start`, `content_block_start`, `content_block_delta` (`text_delta`/`input_json_delta`), `content_block_stop`, `message_delta`, `message_stop`, `ping`.
- Erros: `{ type: "error", error: { type, message } }`; HTTP 400/401/403/404/413/429/500/529.

### OpenAI Chat Completions API (`POST /v1/chat/completions`)
- Headers: `Authorization: Bearer <key>`, `Content-Type: application/json`.
- Request mínimo: `{ model, messages: [{role, content}], max_tokens?, temperature?, stream? }`.
- Response: `{ id, object: "chat.completion", created, model, choices: [{message: {role, content}, finish_reason}], usage: {prompt_tokens, completion_tokens, total_tokens} }`.
- Streaming SSE: `object: "chat.completion.chunk"`, `choices[].delta`, `finish_reason` (null até o fim), terminador `data: [DONE]`.
- Erros: `{ error: { message, type, param, code } }`.

*(Fase 2, ao adicionar mais backends: revalidar cada CLI individualmente quanto a modo headless, formato de auth e suporte a streaming, antes de assumir que a interface `ModelAdapter` é implementável sem gaps.)*

## 11. Estrutura de pastas

```
claude-api-proxy/
├── packages/
│   ├── server/
│   │   ├── src/
│   │   │   ├── routes/            # anthropic.ts, openai.ts
│   │   │   ├── core/               # UnifiedRequest/Response/StreamEvent, Adapter interface
│   │   │   ├── adapters/
│   │   │   │   └── claude-code/
│   │   │   ├── registry/           # carrega/recarrega models.yaml
│   │   │   ├── admin/              # rotas /admin/*
│   │   │   ├── middleware/         # auth, rate-limit, validação
│   │   │   └── config/
│   │   ├── test/
│   │   └── package.json
│   └── desktop/
│       ├── src/                    # main (Electron) + renderer (UI)
│       │   ├── docs/               # markdown offline por API/serviço
│       │   └── ...
│       └── package.json
├── models.yaml
├── .env.example
├── docs/
└── README.md
```

## 12. Testes

- Unit: conversão Anthropic ↔ Unified, OpenAI ↔ Unified, resolução do de-para.
- Integração: `POST /v1/messages` e `POST /v1/chat/completions` (streaming e não-streaming) contra o adapter `claude-code` real (requer sessão local autenticada) ou um fake adapter para CI.
- Contrato: comparar shape da resposta com os schemas oficiais levantados na seção 10.

## 13. Critérios de aceite do MVP

- `POST /v1/messages` e `POST /v1/chat/completions` respondem no formato oficial correspondente, streaming e não-streaming, usando o Claude Code local como motor.
- Trocar o `target_model`/`adapter` de uma entrada em `models.yaml` (via Electron) muda o comportamento sem precisar reiniciar o servidor.
- Electron liga/desliga o servidor, edita credenciais e de-para, e exibe a doc de referência das duas APIs.
- Nenhuma chamada externa é necessária para autenticar — usa a sessão já paga do Claude Code.
