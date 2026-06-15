# ADR-002 — Escolha do LLM: Groq + Llama 4 Scout

> **Data:** 2026-06-04  
> **Atualizado:** 2026-06-15  
> **Status:** ✅ Aceita  
> **Autora:** Julia

---

## Contexto

O sistema precisa de um LLM para gerar respostas conversacionais em tempo real. A latência é crítica em atendimento via WhatsApp — respostas lentas degradam a experiência do lead.

## Decisão

Usar **Groq** como provedor de inferência com o modelo **Llama 4 Scout** (`meta-llama/llama-4-scout-17b-16e-instruct`).

> ⚠️ **Nota:** A documentação inicial mencionava "Llama 3 / Mixtral". O modelo real em uso, conforme extraído dos workflows n8n, é o `meta-llama/llama-4-scout-17b-16e-instruct` (Llama 4 Scout). Corrigido em 2026-06-15.

## Justificativa

| Critério | Groq + Llama 4 Scout | OpenAI GPT-4 | Anthropic Claude |
|----------|---------------------|-------------|-----------------|
| Velocidade (tokens/s) | Muito alta (~500+) | Média (~50) | Média (~60) |
| Custo | Baixo / gratuito em dev | Alto | Médio-alto |
| Qualidade em PT-BR | Boa | Excelente | Excelente |
| Disponibilidade | Alta | Alta | Alta |

A velocidade do Groq é o diferencial principal: respostas em < 1s são essenciais para uma conversa de WhatsApp fluida.

## Consequências

- **Positivo:** Latência muito baixa — experiência de conversa natural
- **Positivo:** Custo reduzido durante desenvolvimento
- **Negativo:** Qualidade em português pode ser inferior ao GPT-4 em casos complexos
- **Risco:** Avaliar se a qualidade das respostas é suficiente para fechamento de vendas

## Revisão

Esta decisão deve ser reavaliada se:
- Qualidade das respostas afetar negativamente a conversão
- Groq alterar sua política de preços significativamente
- Um modelo open source de melhor qualidade em PT-BR se tornar disponível
