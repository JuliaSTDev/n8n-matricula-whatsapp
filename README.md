# 🎓 Agente de Matrículas — WhatsApp + IA + n8n

> Sistema de atendimento comercial autônomo que conduz leads do primeiro contato até a confirmação de matrícula, 100% via WhatsApp, sem operador humano.

---

## O que é

Um conjunto de 7 workflows n8n que automatiza o funil de vendas de uma escola de idiomas. O personagem "Alex" atende, qualifica, apresenta o produto, negocia e fecha matrícula — 24h por dia, respondendo em menos de 2 segundos.

```
Lead envia "Oi" no WhatsApp
        │
        ▼
  Classificador de Intenção (Groq Llama 4 Scout)
        │
   ┌────┴─────────────────────────┐
   ▼                              ▼
[Sondagem]  →  [Pitch]  →  [Negociação]  →  [Matrícula]
   │               │              │
   └───────────────┴──────────────┘
            Retorno agendado (WF6)
```

---

## Stack

| Camada | Tecnologia |
|--------|-----------|
| Orquestração | n8n (self-hosted) |
| LLM — Agentes e Classificador | Groq `meta-llama/llama-4-scout-17b-16e-instruct` |
| Transcrição de áudio | Google Gemini 2.5 Flash |
| Banco de dados | PostgreSQL |
| Memória de conversa | Redis Chat Memory (LangChain) |
| Buffer anti-debounce | Redis (lista, 15s) |
| WhatsApp | Evolution API (Docker) |

---

## Workflows

| # | Nome | Função | Status |
|---|------|--------|--------|
| WF1 | Orquestrador | Webhook de entrada, buffer, classificador, roteamento | 🟡 Em testes |
| WF2 | Sondagem | Qualificação e coleta de dados do lead | 🟡 Em testes |
| WF3 | Pitch | Apresentação do produto personalizada | 🟡 Em testes |
| WF4 | Negociação | Valores, condições e forma de pagamento | 🟡 Em testes |
| WF5 | Matrícula | Confirmação de dados e encerramento | 🟡 Em testes |
| WF6 | Retorno_mensagem | Agendamento de retorno + scheduler automático | 🟡 Em testes |
| WF7 | Simulação | Placeholder de integração com CRM/pagamento | 🔴 Não integrado |

---

## Como funciona

### Fluxo principal

1. Lead manda mensagem no WhatsApp → Evolution API dispara webhook para o n8n
2. Mensagens são bufferizadas por 15s (anti-debounce) e unificadas
3. Classificador de intenção (Groq) decide qual agente responde
4. Agente especialista executa e retorna `{ resposta: string }`
5. Resposta é dividida em partes de ≤ 150 caracteres com 4s de intervalo
6. Evolution API envia cada parte como mensagem separada

### Cross-phase routing

O classificador pode rotear para qualquer agente independente da fase atual do lead. Ex: lead em Sondagem pergunta o preço → vai direto para Negociação.

### Memória compartilhada

Todos os agentes usam o mesmo Redis Chat Memory (session key = `from_id`). O contexto da conversa é preservado quando o lead muda de agente.

---

## Banco de dados

```
leads          → dados do lead + fase atual + metadados (JSONB)
remarcacoes    → agendamentos de retorno com ciclo: aguardando_data → pendente → concluido
unidades       → unidades/sedes da escola (consultado pelo agente de Sondagem)
interacoes     → log de retomadas automáticas feitas pelo scheduler
```

---

## Pré-requisitos

- n8n self-hosted
- PostgreSQL
- Redis
- Evolution API (Docker)
- Conta Groq (API key)
- Conta Google AI (API key para Gemini)

---

## Setup

### 1. Banco de dados

Execute as migrations em `infra/schema.sql` (se disponível) ou crie as tabelas manualmente conforme `docs/integracoes/stack.md`.

### 2. Variáveis de ambiente no n8n

| Variável | Uso |
|----------|-----|
| `EVOLUTION_API_KEY` | Autenticação na Evolution API |
| Credential `Postgres account` | Conexão com PostgreSQL |
| Credential `Redis account` | Conexão com Redis |
| Credential `Groq account` | LLM para agentes |
| Credential `Groq account 2` | LLM para classificador |
| Credential `Google Gemini(PaLM) Api account` | Transcrição de áudio |

> ⚠️ **Nunca commitar credenciais no repositório.** Configure tudo como credentials do n8n ou variáveis de ambiente.

### 3. Importar workflows

Importe os arquivos JSON da pasta `workflows/` na interface do n8n (Settings → Import).

### 4. Tabela `unidades`

Popular a tabela com as unidades reais do cliente antes de ativar em produção:

```sql
INSERT INTO unidades (nome_exibicao, codigo_ps_unidade)
VALUES ('Unidade Centro', 'CI-001'), ('Unidade Sul', 'CI-002');
```

### 5. Ativar webhooks

Ativar o WF1 (Orquestrador) no n8n. O webhook estará em `/webhook-whasapp`. Configurar a Evolution API para enviar eventos de mensagem para essa URL.

---

## Documentação

| Documento | Conteúdo |
|-----------|----------|
| [`docs/arquitetura.md`](docs/arquitetura.md) | Arquitetura técnica completa, fluxo de dados, bugs conhecidos |
| [`docs/fluxos/visao-geral.md`](docs/fluxos/visao-geral.md) | Detalhamento de cada workflow com sequência de nodes |
| [`docs/ia/agentes.md`](docs/ia/agentes.md) | Especificação completa de cada agente: ferramentas, regras, SQL |
| [`docs/ia/prompts.md`](docs/ia/prompts.md) | Histórico de todas as versões de prompt + análise de alucinações |
| [`docs/integracoes/stack.md`](docs/integracoes/stack.md) | Integrações: Evolution API, PostgreSQL, Redis, Groq, Gemini |
| [`docs/testes/casos.md`](docs/testes/casos.md) | Casos de teste organizados por categoria |
| [`docs/testes/log.md`](docs/testes/log.md) | Registro de testes executados |
| [`docs/plano-producao.md`](docs/plano-producao.md) | Checklist e plano para ir a produção |
| [`docs/decisoes/`](docs/decisoes/) | Registros de decisões arquiteturais (ADRs) |

---

## Estrutura do repositório

```
.
├── README.md
├── docs/
│   ├── arquitetura.md          ← visão técnica completa
│   ├── plano-producao.md       ← checklist para ir a produção
│   ├── decisoes/               ← ADRs (por que cada tecnologia foi escolhida)
│   │   ├── 001-evolution-api.md
│   │   ├── 002-llm-groq.md
│   │   └── 003-gemini-audio.md
│   ├── fluxos/
│   │   └── visao-geral.md      ← detalhamento dos 7 workflows
│   ├── ia/
│   │   ├── agentes.md          ← especificação dos agentes
│   │   └── prompts.md          ← versões de prompt + análise
│   ├── integracoes/
│   │   └── stack.md            ← APIs, banco, Redis
│   └── testes/
│       ├── casos.md            ← cenários de teste
│       └── log.md              ← registro de execuções
├── estrategia/
│   └── guia-aaa-js-solucoes.md ← metodologia comercial
├── workflows/                  ← JSONs dos workflows n8n (exportar do n8n)
```

---
