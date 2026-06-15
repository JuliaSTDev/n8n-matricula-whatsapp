# ADR-001 — Escolha da API WhatsApp: Evolution API

> **ADR** = Architectural Decision Record — registro de decisão técnica  
> **Data:** 2026-06-04  
> **Status:** ✅ Aceita  
> **Autora:** Julia

---

## Contexto

O sistema precisa enviar e receber mensagens via WhatsApp de forma programática. Existem três principais opções no mercado: Evolution API (open source / self-hosted), Z-API (SaaS pago), e Meta Business API (oficial).

## Decisão

Usar a **Evolution API** (self-hosted).

## Justificativa

| Critério | Evolution API | Z-API | Meta Business API |
|----------|--------------|-------|-------------------|
| Custo | Gratuito (self-hosted) | Pago por instância | Pago por conversa |
| Controle | Total | Parcial | Limitado |
| Complexidade de setup | Média | Baixa | Alta |
| Dependência de terceiros | Baixa | Alta | Alta |
| Escalabilidade | Alta | Média | Alta |

A Evolution API oferece custo zero e controle total sobre a infraestrutura, o que é prioritário para um sistema em fase de desenvolvimento e testes. Permite iterar rapidamente sem custo por mensagem.

## Consequências

- **Positivo:** Sem custo operacional de API durante desenvolvimento e testes
- **Positivo:** Controle total sobre configuração e comportamento
- **Negativo:** Exige manutenção da instância própria
- **Negativo:** Não é a API oficial — risco de conta ser bloqueada pelo WhatsApp em escala
- **Risco futuro:** Em produção com volume alto, pode ser necessário migrar para Meta Business API

## Revisão

Esta decisão deve ser reavaliada quando:
- O sistema for colocado em produção com leads reais
- O volume de mensagens ultrapassar [definir threshold]
