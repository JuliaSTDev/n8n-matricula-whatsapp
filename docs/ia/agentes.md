# Documentação dos Agentes de IA

> **Última atualização:** 2026-06-05  
> **Modelo:** Groq — `meta-llama/llama-4-scout-17b-16e-instruct`  
> **Persona:** "Alex" (comum a todos os agentes)  
> **Memória compartilhada:** Redis Chat Memory, session key = `from_id`, TTL = 86400s (24h), janela = 50 mensagens

---

## Visão Geral

```
Lead novo → [Sondagem] → [Pitch] → [Negociação] → [Matrícula]
                │              │           │
                └──────────────┴───────────┘
                     retorno_mensagem (WF6)
                   (se lead quer parar/voltar depois)
```

O classificador do Orquestrador pode rotear para qualquer agente em qualquer momento, independente da fase atual do lead.

---

## Agente 01 — Sondagem (WF2)

**Workflow ID:** `CdXDa64U0fHSEHH1oUv29`  
**Inputs recebidos:** `from_id`, `mensagem`, `saudacao_correta`

### Objetivo
Qualificar o lead e coletar todos os dados necessários de forma conversacional antes de revelar qualquer preço ou avançar para o pitch.

### Dados coletados (obrigatórios)

| Campo | Tipo | Regra especial |
|-------|------|---------------|
| Nome_do_aluno | Texto livre | — |
| CPF_do_Aluno | 11 dígitos | Obrigatório antes de mostrar preço |
| Endereco_do_aluno | Texto livre | — |
| Email_do_aluno | email com @ e domínio | Obrigatório antes de avançar para PITCH |
| Unidade | Confirmada via `retornar_sede` | Nunca inventar |
| Nivel_Academico | Iniciante / Intermediário / Avançado | Autoavaliação |
| Motivacao | Carreira / Viagem / Acadêmico | — |
| Disponibilidade_Horario | Manhã / Tarde / Noite / Sábados | — |
| CPF_do_Responsavel | 11 dígitos | Somente se menor de 18 anos |

### Sequência obrigatória

1. Saudar usando `saudacao_correta` → perguntar se o curso é para si ou dependente
2. Usar o nome do cliente assim que souber
3. Identificar objetivo (Carreira / Viagem / Acadêmico)
4. Identificar nível (Iniciante / Intermediário / Avançado)
5. Unidade de preferência + disponibilidade de horário (pode ser na mesma mensagem)
6. Nome completo, CPF, E-mail e Endereço (pode pedir em blocos naturais)

### Ferramentas disponíveis

**`salvar_progresso`**
```sql
UPDATE leads SET metadados = metadados || $dados_json::jsonb WHERE from_id = $from_id;
```
- Trigger: ao coletar qualquer dado relevante
- Formato: JSON com os campos coletados

**`atualizar_fase`**
```sql
UPDATE leads SET fase_atual = $nova_fase WHERE from_id = $from_id
RETURNING 'TRANSICAO_CONCLUIDA. Agora gere APENAS: "Perfeito! Já tenho tudo..."' AS resultado;
```
- Trigger: somente após chamar `salvar_progresso` com todos os dados
- Valor: `PITCH`
- Mensagem de transição obrigatória: "Perfeito, [Nome]! Já tenho tudo que preciso. \nAgora vou te mostrar o curso ideal para o seu objetivo! 🎯"

**`retornar_sede`**
```sql
SELECT id, nome_exibicao, codigo_ps_unidade FROM unidades WHERE nome_exibicao ILIKE '%$termo%' LIMIT 5;
```
- Trigger: somente quando lead informa bairro ou cidade
- PROIBIDO chamar sem o lead ter informado uma localização

**`retorno_mensagem`**
- Chama WF6 com: whatsapp, fase_atual='sondagem', contexto_resumido, data_retorno
- Trigger: somente se lead disser que não pode agora ou quer continuar depois
- PROIBIDO usar para disponibilidade de horário de AULA

### Regras críticas
- Nunca revelar preço antes de CPF + bairro confirmados
- Nunca avançar sem e-mail válido
- Menor de 18 anos: CPF do responsável é obrigatório
- Nunca inventar unidades

---

## Agente 02 — Pitch (WF3)

**Workflow ID:** `-JJ-zV-QGKjLi2E5Q-Bi2`  
**Inputs recebidos:** `from_id`, `mensagem`  
**Busca no banco:** `SELECT id, from_id, fase_atual, metadados FROM leads WHERE from_id = $1`

### Objetivo
Apresentar a solução educacional, gerar desejo e conduzir o lead a validar o interesse antes do fechamento de preço.

### Contexto carregado do banco
- Nome_do_aluno, Motivacao, Nivel_Academico, Unidade, Disponibilidade_Horario

### Sequência do pitch (um ponto por mensagem)

1. **Validação do Perfil** — confirmar que o objetivo é 100% alcançável
2. **Apresentação do Produto** — curso adequado ao nível e motivação
3. **Diferenciais** — certificado com validade internacional
4. **Prova Social** — metodologia de executivos e estudantes com fluência rápida
5. **Escassez** — vagas limitadas na unidade/horário escolhido

### Personalização por motivação
- Carreira → foco em reuniões e ambiente corporativo
- Viagem → foco em autonomia e segurança internacional
- Acadêmico → [adaptar conforme perfil]

### Ferramentas disponíveis

**`atualizar_fase`**
```sql
UPDATE leads SET fase_atual = $nova_fase WHERE from_id = $from_id;
```
- Trigger: lead valida interesse claramente ("Sim, gostei", "Quero saber os valores", "É isso que preciso")
- Valor: `NEGOCIACAO`

**`retorno_mensagem`**
- Chama WF6 com fase_atual e contexto
- Trigger: lead diz que está ocupado ou quer continuar depois

### Price blindage
Se perguntar preço: "Ótima pergunta! Para te passar o valor com as condições especiais de hoje, preciso confirmar a disponibilidade de vaga na sua unidade. Você prefere manhã, tarde ou noite?"

### Mensagem de transição (após atualizar_fase)
"Perfeito, [Nome]! Já tenho tudo que preciso pra te mostrar o plano ideal. 🎯\nPosso te apresentar agora como vai funcionar o seu curso?"

---

## Agente 03 — Negociação (WF4)

**Workflow ID:** `pGo3F8A6xdodmMKXJoQhV`  
**Inputs recebidos:** `from_id`, `mensagem`  
**Busca no banco:** `SELECT id, from_id, fase_atual, metadados FROM leads WHERE from_id = $1`

### Objetivo
Apresentar valores, negociar condições e coletar dados de pagamento para confirmar a matrícula.

### Contexto carregado do banco
- Nome_do_aluno, Unidade, Disponibilidade_Horario, CPF_do_Aluno, Endereco_do_aluno

### Protocolo (um passo por mensagem)

1. Apresentar valor da mensalidade e da matrícula
2. Perguntar código promocional ou convênio
3. Perguntar forma de pagamento:
   - Cartão → coletar bandeira (VISA, MAST ou AMEX) + parcelas (até 12x)
   - Boleto ou PIX
4. Se CPF ou Endereço ausentes → coletar agora
5. Confirmação final com resumo completo

### Ferramentas disponíveis

**`atualizar_fase`**
```sql
UPDATE leads SET fase_atual = $nova_fase WHERE from_id = $from_id
RETURNING '...MATRICULA...' AS resultado;
```
- Trigger: lead confirma o resumo final ("Confirmo", "Pode fazer", "Tá bom")
- Valor: `MATRICULA`
- ⚠️ **BUG identificado:** RETURNING message diz "NEGOCIACAO" em vez de "MATRICULA" — corrigir

**`retorno_mensagem`**
- Chama WF6
- Trigger: lead precisa de tempo

### Regras
- Bandeiras aceitas: VISA, MAST, AMEX apenas
- CPF: verificar 11 dígitos, pedir para conferir se incorreto
- Urgência: reforçar que vaga só é garantida após confirmação final

---

## Agente 04 — Matrícula (WF5)

**Workflow ID:** `DLSPIp9aEVKDLJDQj1smT`  
**Inputs recebidos:** `from_id`, `mensagem`  
**Busca no banco:** `SELECT id, from_id, fase_atual, metadados FROM leads WHERE from_id = $1`

### Objetivo
Confirmar dados, gerar pré-matrícula e enviar link de pagamento por e-mail.

### Dados carregados
Nome, CPF, Email, Endereço, Unidade, Nível, Disponibilidade de Horário

### Sequência obrigatória

1. Apresentar resumo de todos os dados (incluindo e-mail)
2. Se e-mail vazio → coletar antes de prosseguir
3. Perguntar: "Está tudo certo? Posso gerar sua matrícula e o link de pagamento?"
4. Se lead confirmar → chamar `processar_matricula` imediatamente
5. Se lead corrigir dado → voltar ao passo 1
6. Após retorno de `processar_matricula` → enviar mensagem final

### Ferramentas disponíveis

**`processar_matricula`**
- Chama WF7 (Simulação de Matrícula) com: from_id, Nome_do_aluno, Email_do_aluno
- ⚠️ **Status:** Workflow ID é placeholder (`MATRICULA_SIMULADA_ID`) — não funcional em produção
- Retorna: protocolo, link de pagamento, status

### Mensagem final (após `processar_matricula`)
"💳 Tudo certo, [Nome]! Sua pré-matrícula foi registrada (protocolo [PROTOCOLO]).

📧 Acabei de enviar o link de pagamento para o seu e-mail [EMAIL]. Assim que você concluir o pagamento por ele, sua matrícula na Cultura Inglesa estará confirmada e você recebe os dados de acesso e a data de início. Qualquer dúvida, é só me chamar! 😊"

### Proibições absolutas
- NUNCA pedir para ir à unidade presencialmente
- NUNCA oferecer presencial, videochamada ou agendamento de visita
- NUNCA dizer que a matrícula está confirmada antes do pagamento
- NUNCA prometer e-mail sem ter o e-mail coletado

---

## Agente 05 — Retorno_mensagem / Agendamento (WF6)

**Workflow ID:** `lWNhHyvwzJ6v3JOQy0C2G`

Ver documentação completa em `docs/fluxos/visao-geral.md` → Workflow 06.

---

## Tabela de Fases e Transições

| Fase | Agente ativo | Condição de avanço | Próxima fase |
|------|-------------|-------------------|-------------|
| `sondagem` | Agente Sondagem | Todos os dados coletados (incluindo email) | `pitch` |
| `pitch` | Agente Pitch | Lead valida interesse explicitamente | `negociacao` |
| `negociacao` | Agente Negociação | Lead confirma resumo final | `matricula` |
| `matricula` | Agente Matrícula | Lead confirma → `processar_matricula` | — |

**Transições transversais (via classificador do Orquestrador):**
- Qualquer fase → NEGOCIACAO: lead pergunta preço
- Qualquer fase → AGENDAMENTO: lead quer parar e retornar depois

---

## Bugs Documentados

| ID | Agente | Bug | Status |
|----|--------|-----|--------|
| B01 | Negociação | `atualizar_fase` RETURNING diz "NEGOCIACAO" deveria ser "MATRICULA" | ⬜ A corrigir |
| B02 | Matrícula | `processar_matricula` usa ID placeholder — não funciona em produção | ⬜ Aguardando integração real |
| B03 | Sondagem | `contexto_formatado` sempre "Início de conversa" (campo não passado pelo trigger) | ⬜ Investigar se impacta |
