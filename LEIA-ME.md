# Agentes N8N — Processo Comercial Yonzza Digital

Automação completa de prospecção B2B via WhatsApp com qualificação por IA, agendamento automático e supervisão comercial.

---

## Fluxos

| Arquivo | O que faz |
|---|---|
| `fluxo-01-abordagem-diaria.json` | Pega leads novos do Supabase e envia 6 mensagens escalonadas via WhatsApp (seg-sex, 8h) |
| `fluxo-02-recepcao-qualificacao.json` | Recebe respostas, qualifica com SPIN via IA e agenda reunião no Google Calendar |
| `fluxo-03-followup.json` | Envia follow-up automático em D+2, D+3, D+5, D+6 e D+7 para leads sem resposta |
| `fluxo-04-supervisor-comercial.json` | Envia resumo matinal, alerta de tarde, briefing pré-reunião e notificações em tempo real |

---

## Pré-requisitos

- N8N Cloud ativo
- Evolution API conectada — instância `Teste API` online
- Google Calendar OAuth2 configurado no N8N
- Supabase com tabela `leads` e campo `agencia = 'Yonzza Digital'`

---

## Como importar

1. N8N → menu ≡ → **Import from File**
2. Importar na ordem: fluxo-02, 04, 03, 01 (ver seção abaixo)
3. Cada fluxo será criado como **inativo**
4. Configurar credenciais (ver abaixo) antes de ativar

---

## Credenciais necessárias

| Credencial | Usada em | Como configurar |
|---|---|---|
| Evolution API | Fluxos 01, 02, 03, 04 | N8N → Credentials → Evolution API |
| Supabase (HTTP) | Fluxos 01, 02, 03, 04 | Já embutido nos nós HTTP Request |
| Google Calendar OAuth2 | Fluxos 02, 04 | N8N → Credentials → Google Calendar OAuth2 |
| OpenRouter | Fluxos 02, 03, 04 | N8N → Credentials → OpenRouter API |

---

## Configurar webhook no Evolution API (Fluxo 02)

1. Ativar o Fluxo 02 no N8N
2. Copiar a URL do nó **Recebe mensagem**
   - Formato: `https://[seu-n8n].app.n8n.cloud/webhook/whatsapp-yonzza`
3. Evolution API Manager → Instância `Teste API` → **Webhook Settings**
   - URL: colar a URL acima
   - Events: marcar **messages.upsert**
   - Salvar

---

## Ativar na ordem correta

1. **Fluxo 02** — webhook precisa estar ouvindo antes das abordagens
2. **Fluxo 04** — supervisor precisa estar ativo para receber notificações
3. **Fluxo 03** — follow-up automático
4. **Fluxo 01** — dispara na próxima janela de 8h (seg-sex)

---

## Horários automáticos

| Fluxo | Horário (BRT) | Detalhe |
|---|---|---|
| Fluxo 01 — Abordagem | Seg-Sex 8h | Leads `novo`, WhatsApp ativo, até 30/dia, 3 min entre leads |
| Fluxo 02 — Qualificação | Sempre ativo (webhook) | Responde em tempo real. Ignora mensagens `fromMe` e leads já `agendado` |
| Fluxo 03 — Follow-up | Seg-Sex 9h | D+2, D+3, D+5, D+6, D+7 — tracking via `historico_resumido` |
| Fluxo 04 — Resumo | Seg-Sex 8h | Pipeline do dia + reuniões agendadas + bots detectados |
| Fluxo 04 — Alerta | Seg-Sex 17h | Leads travados + abordados sem avanço |
| Fluxo 04 — Briefing | Seg-Sex 8h | Preparação para cada reunião do dia |
| Fluxo 04 — Notificação | Tempo real (webhook) | Reunião agendada ou contato humano necessário |

---

## Marcadores do agente (Fluxo 02)

O agente de IA usa marcadores invisíveis ao lead para acionar ações:

| Marcador | Ação |
|---|---|
| `[[QUALIFICADO]]` | Libera agendamento — deve estar NA MESMA mensagem que os horários |
| `[[AGENDAR:1]]` ou `[[AGENDAR:2]]` | Cria evento no Google Calendar |
| `[[EMAIL_CAPTURADO:email]]` | Salva e-mail no Supabase |
| `[[DESQUALIFICADO]]` | Encerra conversa e atualiza status |
| `[[AGUARDANDO_DECISOR]]` | Marca lead como aguardando decisor |
| `[[CONTATO_HUMANO]]` | Notifica Sara para contato manual |

---

## Status do lead no Supabase

```
novo
 ↓ Fluxo 01 envia abordagem
abordado
 ↓ lead responde → Fluxo 02
qualificando
 ↓ agente identifica ICP
qualificado
 ↓ lead escolhe horário
agendado
 ↓ sem resposta → Fluxo 03
sem_resposta

Status especiais:
aguardando_contato  → lead não é decisor, dados coletados para contato manual
aguardando_decisor  → aguardando indicação do responsável
desqualificado      → fora do perfil
```

> Leads com status `agendado` são ignorados pelo Fluxo 02 — não entram no agente.

---

## Follow-up — lógica de dias (Fluxo 03)

O Fluxo 03 usa `historico_resumido` para controlar quais dias já foram enviados, evitando reenvio mesmo que o `updated_at` do Supabase seja resetado por um PATCH.

| Etapa | Timing | Referência |
|---|---|---|
| D+2 | 2 dias após abordagem | `updated_at` original + sem histórico |
| D+3 | 1 dia após D+2 | `updated_at` pós-D2 |
| D+5 | 2 dias após D+3 | `updated_at` pós-D3 |
| D+6 | 1 dia após D+5 | `updated_at` pós-D5 |
| D+7 | 1 dia após D+6 | `updated_at` pós-D6 → status `sem_resposta` |

---

## Notificações em tempo real (Fluxo 04)

| Tipo | Quando | Formato |
|---|---|---|
| 📅 Reunião agendada | Imediato ao agendar | Nome · Segmento · Horário · Link Meet |
| 🚨 Contato humano | Imediato ao coletar contato | Empresa · WhatsApp do decisor · Histórico · Motivo |
| 🤖 Bot detectado | **Não notifica** — aparece no resumo diário | — |

---

## Resumo diário — formato (Fluxo 04)

```
📊 RESUMO DO DIA
🆕 Novo: X
📨 Abordado: X
💬 Qualificando: X
✅ Qualificado: X
📅 Agendado: X
😶 Sem resposta: X
🤖 Bots detectados: X

📅 REUNIÕES AGENDADAS HOJE
[lista]

💡 OBSERVAÇÕES
[nichos abordados, melhor taxa por nicho, melhor abordagem]
```

---

## Conexões de infraestrutura

- **Supabase URL**: `https://blwkfqfcwlxnydliucrd.supabase.co`
- **Evolution API**: `https://domesticram-evolution.cloudfy.live`
- **Evolution Instance**: `Teste API`
- **N8N Webhook Reunião**: `https://domesticram-n8n.cloudfy.live/webhook/reuniao-agendada-yonzza`

> Se as chaves do Supabase rotacionarem, atualizar nos nós HTTP Request de cada fluxo.

---

## Regras importantes

1. **Nunca usar Code node com `setTimeout` longo** — bloqueia o runner do N8N e causa timeout nos outros fluxos. Usar sempre **Wait node** (`timeInterval`) para esperas acima de 5s.
2. **`updated_at` reseta a cada PATCH no Supabase** — o Fluxo 03 usa `historico_resumido` como proteção contra reenvio de follow-up.
3. **Ordem de importação importa** — Fluxo 02 primeiro (webhook), depois 04, 03, 01.
4. **Somente o AGENTE RESUMO MANHA (Fluxo 04) pode usar emojis** — os demais agentes comunicam com leads e nunca usam emojis.
