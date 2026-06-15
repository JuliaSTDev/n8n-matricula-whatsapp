# Arquitetura Técnica — Sistema de Matrícula Inteligente

> **Versão:** 3.0  
> **Data:** 2026-06-05  
> **Autora:** Julia  
> **Status:** Em desenvolvimento / testes ativos  
> **Fonte:** Extraído de todos os workflows n8n (Orquestrador + 5 agentes)

---

## Visão Geral

Sistema de atendimento conversacional via WhatsApp que automatiza o funil de vendas de matrículas escolares. O personagem "Alex" conduz o lead por 4 fases: Sondagem → Pitch → Negociação → Matrícula. A cada mensagem, um **classificador de intenções** decide qual agente deve responder — permitindo transições dinâmicas (ex: lead em sondagem pergunta preço → vai direto para Negociação).

---

## Stack Tecnológica

| Camada | Tecnologia | Detalhe |
|--------|-----------|---------|
| Orquestração | n8n (self-hosted) | 7 workflows (6 ativos + 1 simulação) |
| IA — Todos os agentes | Groq | Modelo: `meta-llama/llama-4-scout-17b-16e-instruct` |
| IA — Transcrição de áudio | Google Gemini 2.5 Flash | Audio base64 → texto |
| Banco de dados | PostgreSQL | 4 tabelas: leads, remarcacoes, unidades, interacoes |
| Memória de conversa | Redis Chat Memory (LangChain) | Session = from_id, TTL 24h, janela de 50 mensagens |
| Buffer de mensagens | Redis (lista) | Anti-debounce de 15s por chat |
| WhatsApp | Evolution API | Self-hosted Docker: `http://evolution-go:8080` |
| Lógica de negócio | JavaScript (Code node) | Fuso horário, parsing de datas, split de mensagens |

---

## Arquitetura: Fluxo Completo

```
[WhatsApp — usuário envia mensagem]
         │
         ▼
[Evolution API → Webhook n8n POST /webhook-whasapp]
         │
         ▼
[Switch: tipo da mensagem]
    ├── "text"  → extrai .Message.conversation
    └── "media" → base64 → Gemini 2.5 Flash (transcrição)
         │ (Merge)
         ▼
[Redis PUSH buffer_{chat}] → [Wait 15s] → [Redis GET buffer]
         │
         ▼
[IF: última mensagem mudou durante os 15s?]
    ├── SIM  → NoOp (descarta — outra execução vai processar)
    └── NÃO  → continua
         │
         ▼
[Redis DELETE buffer] → [join mensagens → mensagemFinal]
         │
         ▼
[PostgreSQL: SELECT id, from_id, fase_atual FROM leads WHERE from_id LIKE '%Sender%']
         │
         ▼
[IF: Lead existe?]
    ├── SIM → SELECT * leads → Redis GET contexto → data_hora JS → data_mapping_lead
    └── NÃO → INSERT leads (fase='sondagem') → data_hora JS → data_mapping_novo_lead
         │
         ▼ (ambos convergem)
[Classificador de Intenções — Groq Llama 4 Scout]
 Saída: SONDAGEM | PITCH | NEGOCIACAO | AGENDAMENTO | MATRICULA
         │
         ▼
[Switch: roteia por intenção]
    ├── SONDAGEM   → WF2: Sondagem   → resposta (campo "resposta")
    ├── PITCH      → WF3: Pitch       → resposta
    ├── NEGOCIACAO → WF4: Negociação  → resposta
    ├── MATRICULA  → WF5: Matrícula   → resposta
    └── AGENDAMENTO → [fluxo interno Orquestrador]
         │ (todos convergem)
         ▼
[JS: split resposta em partes ≤ 150 chars]
[Loop: Wait 4s → POST Evolution API /send/text/]
[Respond to Webhook]
```

---

## Agentes Especialistas

Todos os agentes compartilham:
- **Persona:** "Alex", vendedor experiente da Cultura Inglesa
- **Modelo:** Groq Llama 4 Scout (`meta-llama/llama-4-scout-17b-16e-instruct`)
- **Memória:** Redis Chat Memory com chave `from_id` — todos os agentes acessam o **mesmo histórico de conversa**
- **Output:** campo `resposta` no JSON de retorno

### WF2 — Sondagem

**Inputs:** `from_id`, `mensagem`, `saudacao_correta`

**Sequência obrigatória:**
1. Saudar (usando `saudacao_correta`) → perguntar se é para si ou dependente
2. Identificar objetivo: Carreira | Viagem | Acadêmico
3. Identificar nível: Iniciante | Intermediário | Avançado
4. Perguntar unidade (bairro/cidade) + disponibilidade de horário (pode ser na mesma mensagem)
5. Coletar: Nome completo, CPF, E-mail, Endereço

**Dados salvos no `leads.metadados` (JSONB):**

| Campo | Observação |
|-------|-----------|
| Nome_do_aluno | |
| CPF_do_Aluno | 11 dígitos |
| Endereco_do_aluno | |
| Email_do_aluno | Obrigatório — link de pagamento vai para aqui |
| Unidade | Confirmada via `retornar_sede` |
| Nivel_Academico | Iniciante / Intermediário / Avançado |
| Motivacao | Carreira / Viagem / Acadêmico |
| Disponibilidade_Horario | Manhã / Tarde / Noite / Sábados |
| CPF_do_Responsavel | Apenas se menor de 18 anos |

**Ferramentas (Tools):**

| Ferramenta | Ação | Trigger |
|-----------|------|---------|
| `salvar_progresso` | `UPDATE leads SET metadados = metadados \|\| $json::jsonb` | Ao coletar dados |
| `atualizar_fase` | `UPDATE leads SET fase_atual = 'PITCH'` | Após salvar_progresso com todos os dados |
| `retornar_sede` | `SELECT id, nome_exibicao, codigo_ps_unidade FROM unidades WHERE ILIKE '%bairro%'` | Quando lead informa bairro/cidade |
| `retorno_mensagem` | Chama WF6 (retorno_mensagem) | Se lead quer continuar depois |

**Regras críticas:**
- Nunca revelar preço antes de coletar CPF + bairro
- Nunca avançar para PITCH sem e-mail válido
- Menor de 18 anos: obrigatório CPF do responsável financeiro
- Nunca inventar unidades — usar somente o que `retornar_sede` retornar
- `retorno_mensagem` é para remarcar atendimento, NÃO para horário de aula

**Transição:** `salvar_progresso` → `atualizar_fase('PITCH')` → mensagem exata: "Perfeito, [Nome]! Já tenho tudo que preciso. \nAgora vou te mostrar o curso ideal para o seu objetivo! 🎯"

---

### WF3 — Pitch

**Inputs:** `from_id`, `mensagem`  
**Busca no banco:** `SELECT id, from_id, fase_atual, metadados FROM leads WHERE from_id = $1`  
**Variáveis de contexto:** Nome_do_aluno, Motivacao, Nivel_Academico, Unidade, Disponibilidade_Horario

**Sequência:**
1. Validação do Perfil — confirmar que o objetivo é alcançável
2. Apresentação do Produto — curso adequado ao nível e motivação
3. Diferenciais — certificado com validade internacional
4. Prova Social — metodologia usada por executivos/estudantes
5. Escassez — vagas limitadas na unidade/horário escolhido

**Ferramentas:**

| Ferramenta | Ação | Trigger |
|-----------|------|---------|
| `atualizar_fase` | `UPDATE leads SET fase_atual = 'NEGOCIACAO'` | Lead valida interesse claramente |
| `retorno_mensagem` | Chama WF6 | Lead quer continuar depois |

**Personalização por motivação:**
- Carreira → foco em reuniões e ambiente corporativo
- Viagem → foco em autonomia e segurança internacional

**Price blindage:** "Para te passar o valor com as condições especiais de hoje, preciso confirmar a disponibilidade de vaga na sua unidade..."

---

### WF4 — Negociação

**Inputs:** `from_id`, `mensagem`  
**Busca no banco:** mesma query do Pitch

**Protocolo (um passo por mensagem):**
1. Apresentar valor da mensalidade e matrícula
2. Perguntar código promocional ou convênio
3. Forma de pagamento: Cartão (VISA/MAST/AMEX, até 12x), Boleto ou PIX
4. Verificar/coletar CPF e Endereço se ausentes
5. Confirmação final com resumo completo

**Ferramentas:**

| Ferramenta | Ação | Trigger |
|-----------|------|---------|
| `atualizar_fase` | `UPDATE leads SET fase_atual = 'MATRICULA'` | Lead confirma resumo final |
| `retorno_mensagem` | Chama WF6 | Lead precisa de tempo |

**Bandeiras aceitas:** VISA, MAST, AMEX (apenas estas)

---

### WF5 — Matrícula

**Inputs:** `from_id`, `mensagem`  
**Dados mapeados:** Nome, CPF, Email, Endereço, Unidade, Nível, Disponibilidade

**Como funciona:**
1. Apresenta resumo completo dos dados coletados
2. Se e-mail ausente → coleta agora
3. Pergunta: "Posso gerar sua matrícula e o link de pagamento?"
4. Se confirmado → chama `processar_matricula`
5. Informa que link foi enviado para o e-mail
6. Matrícula fica PENDENTE até o pagamento ser efetuado

**Ferramentas:**

| Ferramenta | Ação | Trigger |
|-----------|------|---------|
| `processar_matricula` | Chama WF7 (simulação) | Após confirmação explícita do lead |

**Proibições absolutas:**
- NUNCA pedir para ir à unidade presencialmente
- NUNCA oferecer visita ou videochamada
- NUNCA confirmar matrícula antes do pagamento
- NUNCA prometer enviar e-mail sem ter o e-mail coletado

---

### WF6 — Retorno_mensagem (Agendamento)

**Dois modos de operação:**

**Modo 1 — Chamado como sub-workflow pelos agentes:**
- Recebe: whatsapp, data_retorno, fase_atual, fase_atendimento, contexto_resumido, status
- INSERT INTO remarcacoes (status='pendente')
- Retorna: `{ status: 'sucesso', mensagem: 'O retorno foi agendado com sucesso' }`

**Modo 2 — Scheduler autônomo (trigger a cada 1 minuto):**
```
[Schedule: toda minuto]
    → SELECT FROM remarcacoes WHERE status='pendente' AND data_retorno <= NOW() AT TIME ZONE 'America/Sao_Paulo'
    → IF lead existe:
        → POST Evolution API: "Oi! Conforme combinamos, estou passando para continuarmos nosso contato..."
        → INSERT INTO interacoes (mensagem_usuario=contexto_resumido, resposta_ia='SISTEMA: Retomada Automática')
        → UPDATE remarcacoes SET status='concluido'
    → IF não existe: NoOp
```

**Ciclo de vida do status em `remarcacoes`:**

```
[agente detecta AGENDAMENTO]
    ├── Se já existe registro 'aguardando_data' → extrai data da mensagem → status = 'pendente'
    └── Se não existe → INSERT status = 'aguardando_data' → pergunta a data
                                         ↓
                               Lead informa a data
                                         ↓
                              status = 'pendente'
                                         ↓
                         Scheduler dispara no horário
                                         ↓
                              status = 'concluido'
```

---

### WF7 — Simulação de Matrícula (Placeholder)

> ⚠️ **Status: EM DESENVOLVIMENTO** — Não conectado a sistema real

**Função atual:** Gera protocolo e link de pagamento simulados para testes.

**Lógica:**
```javascript
protocolo = `CI-${ano}-${6 dígitos aleatórios}`
link = `https://pagamento.culturainglesa.com/p/${protocolo}`
```

**Saída:**
```json
{
  "status": "pre_matricula_registrada",
  "protocolo": "CI-2026-384921",
  "link_pagamento": "https://pagamento.culturainglesa.com/p/CI-2026-384921",
  "email_destino": "email@do.lead",
  "resultado": "Pre-matricula registrada..."
}
```

**Integração futura:** Este workflow deve ser substituído pela integração real com o CRM Oracle e/ou gateway de pagamento.

---

## Banco de Dados — Schema Completo

### `leads`
```sql
CREATE TABLE leads (
  id         SERIAL PRIMARY KEY,
  from_id    VARCHAR NOT NULL,     -- "5511999999999@s.whatsapp.net"
  fase_atual VARCHAR DEFAULT 'sondagem',  -- sondagem | pitch | negociacao | matricula
  metadados  JSONB   DEFAULT '{}'
);
-- metadados contém: Nome_do_aluno, CPF_do_Aluno, Endereco_do_aluno, Email_do_aluno,
--                   Unidade, Nivel_Academico, Motivacao, Disponibilidade_Horario,
--                   CPF_do_Responsavel (se menor)
```

### `remarcacoes`
```sql
CREATE TABLE remarcacoes (
  id                 SERIAL PRIMARY KEY,
  whatsapp           VARCHAR NOT NULL,
  data_retorno       TIMESTAMP,      -- UTC
  fase_atual         VARCHAR,
  fase_atendimento   VARCHAR,
  contexto_resumido  TEXT,
  status             VARCHAR,        -- 'aguardando_data' | 'pendente' | 'concluido'
  criado_em          TIMESTAMP DEFAULT NOW()
);
```

### `unidades`
```sql
-- Consultada pelo Agente Sondagem (retornar_sede)
CREATE TABLE unidades (
  id                  SERIAL PRIMARY KEY,
  nome_exibicao       VARCHAR,       -- buscado por ILIKE '%bairro%'
  codigo_ps_unidade   VARCHAR        -- código interno da unidade
);
```

### `interacoes`
```sql
-- Registrada pelo WF6 (Agendamento) a cada retomada automática
CREATE TABLE interacoes (
  id               SERIAL PRIMARY KEY,
  lead_id          INTEGER,
  mensagem_usuario TEXT,
  resposta_ia      TEXT,
  fase_no_momento  VARCHAR,
  data_interacao   TIMESTAMP
);
```

---

## Redis — Estrutura de Chaves

| Chave | Tipo | Conteúdo | TTL |
|-------|------|----------|-----|
| `buffer_{chat_jid}` | Lista | Buffer anti-debounce 15s | Manual (DEL após processar) |
| `{fase_atual}_{from_id}` | Valor | Cache contexto (Orquestrador) | [verificar] |
| Chave Redis Chat Memory | Gerenciada pelo LangChain | Histórico de 50 mensagens por lead | 86400s (24h) |

> **Importante:** Todos os agentes (Sondagem, Pitch, Negociação, Matrícula) usam a **mesma chave de sessão** (`from_id`) no Redis Chat Memory. Isso garante que o contexto da conversa é preservado entre trocas de agente.

---

## Mapa de Workflows

| # | Nome | Workflow ID | Tipo | Status |
|---|------|------------|------|--------|
| 1 | Orquestrador | (principal) | Webhook | 🟡 Em testes |
| 2 | Sondagem | `CdXDa64U0fHSEHH1oUv29` | Sub-workflow | 🟡 Em testes |
| 3 | Pitch | `-JJ-zV-QGKjLi2E5Q-Bi2` | Sub-workflow | 🟡 Em testes |
| 4 | Negociação | `pGo3F8A6xdodmMKXJoQhV` | Sub-workflow | 🟡 Em testes |
| 5 | Matrícula | `DLSPIp9aEVKDLJDQj1smT` | Sub-workflow | 🟡 Em testes |
| 6 | Retorno_mensagem | `lWNhHyvwzJ6v3JOQy0C2G` | Sub-workflow + Scheduler | 🟡 Em testes |
| 7 | Simulação Matrícula | `MATRICULA_SIMULADA_ID` (placeholder) | Sub-workflow | 🔴 Não integrado |

---

## ⚠️ Bugs e Pontos de Atenção Identificados

### Bugs confirmados

| # | Local | Descrição | Impacto |
|---|-------|-----------|---------|
| B01 | WF4 Negociação — `atualizar_fase` | RETURNING message diz "NEGOCIACAO" quando deveria dizer "MATRICULA" (copy-paste do Pitch) | Médio — pode confundir o LLM sobre o que fazer após a transição |
| B02 | WF5 Matrícula — `processar_matricula` | Usa `MATRICULA_SIMULADA_ID` como ID literal — não é o ID real do workflow | Alto — vai falhar em produção |
| B03 | Orquestrador vs Agentes | Orquestrador chama Retorno_mensagem com ID `GuT37rAjec6IphwMpV9Ff`; agentes usam `lWNhHyvwzJ6v3JOQy0C2G` — IDs diferentes | Médio — verificar se são o mesmo workflow |
| B04 | WF2 Sondagem — `contexto_formatado` | Referencia `$json.propertyName` mas o trigger não passa este campo — sempre retorna "Início de conversa" | Baixo — contexto via Redis Chat Memory compensa |

### Riscos técnicos

| # | Risco | Mitigação sugerida |
|---|-------|-------------------|
| R01 | `WHERE from_id LIKE '%Sender%'` sem índice — lento em tabela grande | Criar índice ou mudar para `=` |
| R02 | Scheduler do WF6 roda a cada 1 minuto — pode gerar carga desnecessária | Aumentar intervalo ou usar trigger baseado em evento |
| R03 | Redis Chat Memory sem TTL separado — se TTL de 24h expirar no meio de uma negociação, perde contexto | Monitorar e avaliar aumentar TTL |
| R04 | API key da Evolution exposta no JSON dos workflows | Mover para variável de ambiente n8n |
| R05 | Simulação de matrícula gera link falso (`pagamento.culturainglesa.com`) — se enviado para lead real, causa problema | Garantir que WF7 nunca seja chamado em produção antes da integração real |

---

## Decisões Técnicas

| Decisão | Escolha | Ver |
|---------|---------|-----|
| API WhatsApp | Evolution API (Docker interno) | `docs/decisoes/001-evolution-api.md` |
| LLM | Groq Llama 4 Scout | `docs/decisoes/002-llm-groq.md` |
| Transcrição de áudio | Gemini 2.5 Flash | `docs/decisoes/003-gemini-audio.md` |
| Memória compartilhada entre agentes | Redis Chat Memory (mesma session key) | — |
| Classificador antes dos agentes | Permite cross-fase (ex: sondagem → negociação direta) | — |

---

## Histórico de Versões deste Documento

| Versão | Data | Alteração |
|--------|------|-----------|
| 1.0 | 2026-06-04 | Criação inicial |
| 2.0 | 2026-06-05 | Reescrito com dados do Orquestrador |
| 3.0 | 2026-06-05 | Adicionados todos os 7 workflows, bugs identificados, schema completo |
| 3.1 | 2026-06-15 | Corrigida referência ao modelo LLM (Llama 3 → Llama 4 Scout); revisão geral para publicação no GitHub |
