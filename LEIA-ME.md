# Fluxos N8N — Processo Comercial Yonzza Digital

## Arquivos

| Arquivo | O que faz |
|---|---|
| `fluxo-01-abordagem-diaria.json` | Pega leads novos do Supabase e envia mensagem inicial via WhatsApp (seg-sex, 8h) |
| `fluxo-02-recepcao-qualificacao.json` | Recebe respostas do WhatsApp, qualifica o lead em 3 perguntas e agenda a call |
| `fluxo-03-followup.json` | Envia follow-up automático em D+2, D+5 e D+7 para leads sem resposta |

---

## Pré-requisitos

- [ ] N8N Cloud ativo
- [ ] Evolution API conectada e instância "Teste API" online
- [ ] Google Calendar — conta Google conectada no N8N
- [ ] Supabase com coluna `mensagem_wa` criada (ver abaixo)

---

## Passo 1 — Adicionar coluna mensagem_wa no Supabase

Rodar no SQL Editor do Supabase:

```sql
ALTER TABLE leads ADD COLUMN IF NOT EXISTS mensagem_wa text;
```

Depois rodar `fase_sync_supabase.py` para sincronizar os dados atuais.

---

## Passo 2 — Importar os fluxos no N8N

1. Abrir N8N → Menu → **Import from File**
2. Importar na ordem: fluxo-01, fluxo-02, fluxo-03
3. Cada fluxo será criado como **inativo** — não ative ainda

---

## Passo 3 — Configurar Google Calendar (Fluxo 02)

1. No N8N → **Credentials** → **Add Credential** → **Google Calendar OAuth2**
2. Nomear como `Google Calendar Yonzza`
3. Autorizar com a sua conta Google
4. No fluxo 02, encontrar o nó **"Criar Evento Google Calendar"**
5. Substituir `SEU_GOOGLE_CALENDAR_ID_AQUI` pelo ID da sua agenda:
   - Google Calendar → Configurações da agenda → **ID do calendário**
   - ✅ Já configurado: `agenciayonzza@gmail.com`

---

## Passo 4 — Configurar Webhook no Evolution API (Fluxo 02)

1. Ativar o fluxo 02 no N8N
2. Abrir o nó **"Webhook Evolution API"** → copiar a **Webhook URL**
   - Formato: `https://[seu-n8n].app.n8n.cloud/webhook/whatsapp-yonzza`
3. No Evolution API Manager (`https://domesticram-evolution.cloudfy.live/manager`):
   - Instância "Teste API" → **Webhook Settings**
   - URL: colar a URL do passo 2
   - Events: marcar **messages.upsert**
   - Salvar

---

## Passo 5 — Ativar os fluxos na ordem

1. ✅ Ativar **Fluxo 02** primeiro (o webhook precisa estar ouvindo)
2. ✅ Ativar **Fluxo 03** (follow-up automático)
3. ✅ Ativar **Fluxo 01** por último (vai disparar na próxima janela de 8h)

---

## Fluxo de status dos leads

```
novo
 ↓ (Fluxo 01 envia mensagem)
abordado
 ↓ (lead responde → Fluxo 02)
qual_1 → (tem agência?) → desqualificado
 ↓ (não tem)
qual_2 → (não é decisor) → aguarda indicação
 ↓ (é decisor)
qual_3 → (segmento bloqueado) → desqualificado
 ↓ (segmento OK)
agendando → (escolhe horário) → agendado
 ↓ (sem resposta após D+7, Fluxo 03)
sem_resposta
```

---

## Horários de execução

| Fluxo | Horário | Observação |
|---|---|---|
| Fluxo 01 — Abordagem | Seg-Sex 8h | Máx 10 leads/dia, intervalo 3min entre envios |
| Fluxo 02 — Qualificação | Sempre ativo (webhook) | Responde em tempo real |
| Fluxo 03 — Follow-up | Seg-Sex 9h | D+2, D+5, D+7 desde o dia da abordagem |

---

## Nichos bloqueados (configurados nos fluxos 01 e 02)

Fluxo 01 (filtro de extração): `bet, aposta, crypto, criptomoeda, loteria, casino, cassino, jogo online, gaming`

Fluxo 02 (Q3 — resposta do lead): mesma lista acima

Para adicionar/remover nichos: editar o nó **"Filtrar Nichos Bloqueados"** (Fluxo 01) e **"Processar Resposta Q3"** (Fluxo 02).

---

## Perguntas de qualificação

| Etapa | Pergunta | Desqualifica se |
|---|---|---|
| Q1 | Já trabalham com alguma agência? | Responder sim |
| Q2 | Você é o responsável pelas decisões de marketing? | Responder não |
| Q3 | Qual o principal serviço/produto do negócio? | Segmento bloqueado |

---

## Credenciais já configuradas nos JSONs

As seguintes credenciais já estão embutidas nos fluxos:

- **Supabase URL**: `https://blwkfqfcwlxnydliucrd.supabase.co`
- **Evolution API URL**: `https://domesticram-evolution.cloudfy.live`
- **Evolution Instance**: `Teste API`

> ⚠️ Se as chaves rotacionarem, atualizar nos nós HTTP Request de cada fluxo.
