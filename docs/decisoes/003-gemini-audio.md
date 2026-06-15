# ADR-003 — Transcrição de Áudio: Google Gemini 2.5 Flash

> **Data:** 2026-06-05  
> **Status:** ✅ Aceita  
> **Autora:** Julia

---

## Contexto

O WhatsApp é amplamente usado com mensagens de áudio no Brasil. O sistema precisa processar áudios dos leads para não perder intenções comunicadas por voz.

## Decisão

Usar **Google Gemini 2.5 Flash** para transcrição de áudio via `@n8n/n8n-nodes-langchain.googleGemini`.

## Justificativa

- Suporte nativo a áudio em português brasileiro
- Integração direta via node LangChain no n8n
- Modelo multimodal — processa áudio diretamente sem pipeline separado
- Custo acessível no nível de uso atual

## Implementação

Fluxo: `Info.Type = "media"` → `converter_audio` (base64 → binário) → `transcrever_audio` (Gemini) → `msg_audio` (extrai `.content.parts[0].text`)

Fallback configurado: se transcrição falhar, continua com `[audio nao transcrito]` (não quebra o fluxo).

## Consequências

- **Positivo:** Sistema funciona com texto e áudio transparentemente
- **Negativo:** Dependência de API externa adicional (Groq + Gemini)
- **Risco:** Custo adicional com volume alto de áudios
- **Pendente:** Testar qualidade da transcrição com sotaques e ruído de fundo
