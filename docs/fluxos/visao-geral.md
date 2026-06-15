# Visão Geral dos Workflows n8n

> **Última atualização:** 2026-06-05  
> **Total de workflows:** 7 (6 ativos + 1 simulação)  
> **Fonte:** Extraído dos JSONs reais de todos os workflows

---

## Mapa de Dependências

```
[WF1: Orquestrador] ← webhook Evolution API
    │
    ├── Buffer Redis (15s) ──────────────────────────────────────────────┐
    │                                                                     │ descarta se nova
    ├── PostgreSQL: verifica/cria lead                                    │ msg chegou
    │                                                                     │
    ├── Classificador Groq (intenção) → SONDAGEM   → [WF2: Sondagem]    │
    │                                 → PITCH       → [WF3: Pitch]       │
    │                                 → NEGOCIACAO  → [WF4: Negociação]  │
    │                                 → MATRICULA   → [WF5: Matrícula]   │
    │                                 → AGENDAMENTO → [lógica interna]───┘
    │                                                      ↓
    │                                               [WF6: Retorno_msg]
    │
    └── Todos os agentes chamam [WF6] quando lead quer parar
    └── WF5 (Matrícula) chama [WF7: Simulação] ao fechar

[WF6: Retorno_mensagem]
    ├── Modo sub-workflow: INSERT remarcacoes (status=pendente)
    └── Modo scheduler (a cada 1 min): dispara mensagem para leads com data_retorno vencida
```

---

## WF1 — Orquestrador

| Campo | Valor |
|-------|-------|
| **Trigger** | Webhook POST `/webhook-whasapp` (Evolution API) |
| **Status** | 🟡 Em testes |
| **Tipo** | Workflow principal — coordena tudo |

**Responsabilidades:**
- Receber e normalizar mensagens (texto e áudio via Gemini)
- Buffer anti-debounce de 15s (Redis)
- Verificar/criar lead no PostgreSQL
- Carregar contexto da conversa (Redis)
- Classificar intenção via Groq Llama 4 Scout
- Rotear para sub-workflow correto
- Gerenciar fluxo de agendamento
- Enviar resposta em partes (≤150 chars, 4s entre cada)

**Sequência de nodes:**
```
WhatsApp (webhook)
  └─► Switch (texto / media)
        ├─► msg_texto (Set: extrai .conversation)
        └─► converter_audio → transcrever_audio (Gemini) → msg_audio
              └─► Merge
                    └─► addBuffer (Redis PUSH) → getBuffer → Wait 15s → getBuffer4
                          └─► IF (debounce check)
                                ├─ TRUE  → deleteBuffer → mensagemFinal (Set: join \n\n)
                                │              └─► Fase_Atual (PG SELECT)
                                │                    └─► Lead Existe? (IF)
                                │                          ├─ SIM → Fase_Atual4 (PG) → contexo (Redis) → data_hora → data_mapping_lead
                                │                          └─ NÃO → Inserir lead novo (PG) → data_hora4 → data_mapping_novo_lead
                                │                                └─► Basic LLM Chain (classificador Groq)
                                │                                      └─► sanitizar_classificacao1 (JS)
                                │                                            └─► Switch4 (roteia)
                                │                                                  ├─► Call 'Sondagem'
                                │                                                  ├─► Call 'Pitch'
                                │                                                  ├─► Call 'Negociacao'
                                │                                                  ├─► Call 'Matricula'
                                │                                                  └─► checar_remarcacao_pendente → ja_aguardando_data?
                                │                                                        ├─ SIM → extrair_data_retorno → parsear → confirmar → Call Retorno_msg
                                │                                                        └─ NÃO → inserir_remarcacao_pendente → resposta_pergunta_horario
                                │                                                └─► Code JS (split resposta) → Loop → Wait 4s → HTTP POST Evolution → Respond Webhook
                                └─ FALSE → No Operation (descarta)
```

---

## WF2 — Sondagem

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `CdXDa64U0fHSEHH1oUv29` |
| **Trigger** | Chamado pelo WF1 (`Execute Workflow`, aguarda resposta) |
| **Inputs** | `from_id`, `mensagem`, `saudacao_correta` |
| **Output** | `{ resposta: string }` |
| **Status** | 🟡 Em testes |
| **Modelo** | Groq Llama 4 Scout (`Groq account`) |
| **Memória** | Redis Chat Memory (session=from_id, TTL=24h, janela=50 msgs) |

**Responsabilidades:**
- Identificar se curso é para o próprio lead ou dependente
- Coletar objetivo, nível, unidade, disponibilidade, dados pessoais e e-mail
- Verificar unidades disponíveis via tabela `unidades`
- Salvar dados no `leads.metadados` (JSONB merge incremental)
- Avançar fase para `pitch` quando todos os dados estiverem coletados

**Ferramentas disponíveis para o agente:**

| Ferramenta | Tipo | O que faz |
|-----------|------|-----------|
| `salvar_progresso` | PostgreSQL Tool | `UPDATE leads SET metadados = metadados \|\| $json` |
| `atualizar_fase` | PostgreSQL Tool | `UPDATE leads SET fase_atual = 'PITCH'` |
| `retornar_sede` | PostgreSQL Tool | `SELECT FROM unidades WHERE ILIKE '%bairro%'` |
| `retorno_mensagem` | Workflow Tool | Chama WF6 para agendar retorno |

**Sequência de nodes:**
```
When Executed by Another Workflow
  └─► data_mapping_sondagem (Set: mapeia contexto + inputs)
        └─► Sondagem (Agent LLM)
              ├── Groq Chat Model (ai_languageModel)
              ├── Redis Chat Memory (ai_memory)
              ├── salvar_progresso (ai_tool)
              ├── atualizar_fase (ai_tool)
              ├── retornar_sede (ai_tool)
              └── retorno_mensagem (ai_tool)
        └─► Code in JavaScript1 (extrai .output → { resposta })
```

---

## WF3 — Pitch

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `-JJ-zV-QGKjLi2E5Q-Bi2` |
| **Trigger** | Chamado pelo WF1 |
| **Inputs** | `from_id`, `mensagem` |
| **Output** | `{ resposta: string }` |
| **Status** | 🟡 Em testes |
| **Modelo** | Groq Llama 4 Scout (`Groq account`) |
| **Memória** | Redis Chat Memory (mesma sessão do WF2) |

**Responsabilidades:**
- Carregar dados do lead do banco (metadados)
- Apresentar proposta personalizada por motivação e nível
- Usar gatilho de escassez (vagas limitadas)
- Avançar fase para `negociacao` quando lead confirmar interesse

**Ferramentas disponíveis:**

| Ferramenta | Tipo | O que faz |
|-----------|------|-----------|
| `atualizar_fase` | PostgreSQL Tool | `UPDATE leads SET fase_atual = 'NEGOCIACAO'` |
| `retorno_mensagem` | Workflow Tool | Chama WF6 |

**Sequência de nodes:**
```
When Executed by Another Workflow
  └─► lead (PG: SELECT * leads WHERE from_id)
        └─► data_mapping (Set: mapeia Nome, Motivacao, Nivel, Unidade, Disponibilidade)
              └─► PITCH (Agent LLM)
                    ├── Groq Chat Model
                    ├── Redis Chat Memory
                    ├── atualizar_fase (ai_tool)
                    └── retorno_mensagem (ai_tool)
              └─► Code in JavaScript1 → { resposta }
```

---

## WF4 — Negociação

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `pGo3F8A6xdodmMKXJoQhV` |
| **Trigger** | Chamado pelo WF1 |
| **Inputs** | `from_id`, `mensagem` |
| **Output** | `{ resposta: string }` |
| **Status** | 🟡 Em testes |
| **Modelo** | Groq Llama 4 Scout (`Groq account`) |
| **Memória** | Redis Chat Memory (mesma sessão) |

**Responsabilidades:**
- Apresentar valores de mensalidade e matrícula
- Verificar código promocional/convênio
- Coletar forma de pagamento (Cartão VISA/MAST/AMEX até 12x, Boleto, PIX)
- Verificar/coletar CPF e endereço se ausentes
- Resumo final para confirmação
- Avançar fase para `matricula` após confirmação

**Ferramentas disponíveis:**

| Ferramenta | Tipo | O que faz |
|-----------|------|-----------|
| `atualizar_fase` | PostgreSQL Tool | `UPDATE leads SET fase_atual = 'MATRICULA'` ⚠️ Bug B01 no RETURNING |
| `retorno_mensagem` | Workflow Tool | Chama WF6 |

**Sequência de nodes:**
```
When Executed by Another Workflow
  └─► lead (PG: SELECT * leads WHERE from_id)
        └─► data_mapping (Set: mapeia todos os campos + contexto_formatado)
              └─► Negociação (Agent LLM)
                    ├── Groq Chat Model
                    ├── Redis Chat Memory
                    ├── atualizar_fase (ai_tool)
                    └── retorno_mensagem (ai_tool)
              └─► Code in JavaScript1 → { resposta }
```

---

## WF5 — Matrícula

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `DLSPIp9aEVKDLJDQj1smT` |
| **Trigger** | Chamado pelo WF1 |
| **Inputs** | `from_id`, `mensagem` |
| **Output** | `{ resposta: string }` |
| **Status** | 🟡 Em testes (com simulação) |
| **Modelo** | Groq Llama 4 Scout (`Groq account`) |
| **Memória** | Redis Chat Memory (mesma sessão) |

**Responsabilidades:**
- Carregar todos os dados coletados
- Verificar se e-mail está presente (coletar se ausente)
- Confirmar dados com o lead
- Chamar `processar_matricula` após confirmação explícita
- Informar que link de pagamento foi enviado por e-mail
- Deixar claro que matrícula só é confirmada após o pagamento

**Ferramentas disponíveis:**

| Ferramenta | Tipo | O que faz |
|-----------|------|-----------|
| `processar_matricula` | Workflow Tool | Chama WF7 (simulação) ⚠️ Bug B02: ID placeholder |

**Sequência de nodes:**
```
When Executed by Another Workflow
  └─► lead1 (PG: SELECT * leads WHERE from_id)
        └─► data_mapping (Set: Nome, CPF, Email, Endereço, Unidade, Nível, Horário)
              └─► Matricula1 (Agent LLM)
                    ├── Groq Chat Model1
                    ├── Redis Chat Memory1
                    └── processar_matricula (ai_tool → WF7)
              └─► Code in JavaScript → { resposta }
```

---

## WF6 — Retorno_mensagem (Agendamento)

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `lWNhHyvwzJ6v3JOQy0C2G` |
| **Triggers** | 2: `executeWorkflowTrigger` (sub-workflow) + `scheduleTrigger` (a cada 1 min) |
| **Status** | 🟡 Em testes |

**Dois modos de operação:**

### Modo 1 — Sub-workflow (chamado pelos agentes)

**Inputs:** `whatsapp`, `data_retorno`, `fase_atual`, `fase_atendimento`, `contexto_resumido`, `status`

```
When Executed by Another Workflow
  └─► insert_remarcacoes (PG: INSERT INTO remarcacoes, status='pendente')
        └─► Edit Fields (Set: { status: 'sucesso', mensagem: 'O retorno foi agendado...' })
```

**Output:** `{ status: 'sucesso', mensagem: 'O retorno foi agendado com sucesso para a data solicitada.' }`

### Modo 2 — Scheduler autônomo (a cada 1 minuto)

```
Schedule Trigger (every 1 minute)
  └─► selecionar_leads_pendentes
        (PG: SELECT * FROM remarcacoes WHERE status='pendente' AND data_retorno <= NOW() AT TIME ZONE 'America/Sao_Paulo')
        └─► existe_lead? (IF: id not empty)
              ├─ SIM → enviar_mensagem_lead
              │         (HTTP POST Evolution: "Oi! Conforme combinamos...")
              │         └─► salvar_interacao
              │               (PG: INSERT INTO interacoes)
              │               └─► update_status
              │                     (PG: UPDATE remarcacoes SET status='concluido')
              └─ NÃO → No Operation
```

**Mensagem enviada na retomada:**
> "Oi! Conforme combinamos, estou passando para continuarmos nosso contato. Como posso te ajudar?"

---

## WF7 — Simulação de Matrícula

| Campo | Valor |
|-------|-------|
| **Workflow ID** | `MATRICULA_SIMULADA_ID` ⚠️ Placeholder — não é ID real |
| **Trigger** | Chamado pelo WF5 |
| **Inputs** | `from_id`, `Nome_do_aluno` (+ `Email_do_aluno` do contexto) |
| **Status** | 🔴 Placeholder — não integrado a sistema real |

**O que faz:**
- Gera protocolo aleatório: `CI-{ano}-{6 dígitos}`
- Gera link de pagamento fake: `https://pagamento.culturainglesa.com/p/{protocolo}`
- Retorna status `pre_matricula_registrada`

**Substituição futura:** Este workflow deve ser integrado ao CRM Oracle e/ou gateway de pagamento real. Até lá, só usar em ambiente de testes.

---

## Glossário de Status dos Workflows

| Ícone | Significado |
|-------|-------------|
| 🟢 | Estável — funcionando conforme esperado em produção |
| 🟡 | Em testes — funcional mas sendo aprimorado |
| 🔴 | Com problema / não integrado |

## Bugs Conhecidos

| ID | Workflow | Descrição |
|----|----------|-----------|
| B01 | WF4 Negociação | `atualizar_fase` RETURNING diz "NEGOCIACAO" em vez de "MATRICULA" |
| B02 | WF5 Matrícula | `processar_matricula` usa ID `MATRICULA_SIMULADA_ID` (literal, não funciona em produção) |
| B03 | WF1 vs WF2-4 | WF1 chama Retorno_mensagem com ID `GuT37rAjec6IphwMpV9Ff`; agentes usam `lWNhHyvwzJ6v3JOQy0C2G` — verificar se são o mesmo |
