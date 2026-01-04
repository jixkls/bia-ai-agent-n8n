# 🤖 Bia - Assistente Virtual de Agendamentos via WhatsApp

Bia é uma secretária virtual inteligente que gerencia agendamentos do Google Calendar através do WhatsApp, utilizando IA (Google Gemini) para interpretar mensagens de texto e áudio em linguagem natural.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Arquitetura](#-arquitetura)
3. [Stack Tecnológica](#-stack-tecnológica)
4. [Pré-requisitos](#-pré-requisitos)
5. [Setup Completo](#-setup-completo)
6. [Schema do Banco de Dados](#-schema-do-banco-de-dados)
7. [Fluxo do Workflow](#-fluxo-do-workflow)
8. [Funcionalidades](#-funcionalidades)
9. [Estrutura JSON da IA](#-estrutura-json-da-ia)
10. [Tratamento de Erros](#-tratamento-de-erros)
11. [Testes Realizados](#-testes-realizados)
12. [Troubleshooting](#-troubleshooting)
13. [Melhorias Futuras](#-melhorias-futuras)

---

## 🎯 Visão Geral

### O que é a Bia?

Bia é um agente conversacional que permite aos usuários gerenciar seus compromissos no Google Calendar através de mensagens no WhatsApp. O sistema processa tanto mensagens de texto quanto áudios, interpretando a intenção do usuário e executando ações no calendário.

### Casos de Uso

- **Criar agendamentos**: "Bia, marca uma reunião com o João amanhã às 14h"
- **Alterar eventos**: "Bia, muda a reunião das 14h para 15h"
- **Cancelar eventos**: "Bia, cancela a reunião com a Maria"
- **Consultar disponibilidade**: "Bia, estou livre amanhã às 10h?"
- **Confirmações inteligentes**: Sistema de confirmação antes de executar ações

### Diferenciais

- ✅ Suporte a áudio (transcrição automática via Gemini)
- ✅ Memória de contexto por conversa (PostgreSQL)
- ✅ Validação de horário comercial (Seg-Sex, 09:00-18:00)
- ✅ Idempotência (evita processamento duplicado)
- ✅ Tratamento robusto de erros com mensagens amigáveis
- ✅ Logs completos para rastreabilidade

---

## 🏗 Arquitetura

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    WhatsApp     │────▶│     Z-API       │────▶│     Webhook     │
│    (Usuário)    │◀────│   (Gateway)     │◀────│     (n8n)       │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                        ┌────────────────────────────────┼────────────────────────────────┐
                        │                                │                                │
                        ▼                                ▼                                ▼
               ┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
               │  Google Gemini  │              │    Supabase     │              │ Google Calendar │
               │  (IA + STT)     │              │   (PostgreSQL)  │              │     (API)       │
               └─────────────────┘              └─────────────────┘              └─────────────────┘
```

### Fluxo Simplificado

1. Usuário envia mensagem (texto/áudio) no WhatsApp
2. Z-API recebe e envia webhook para n8n
3. n8n processa a mensagem:
   - Se áudio: transcreve com Gemini
   - Interpreta intenção com AI Agent
   - Executa ação no Google Calendar
4. Resposta é enviada de volta ao usuário via Z-API

---

## 🛠 Stack Tecnológica

| Componente | Tecnologia | Função |
|------------|------------|--------|
| **Orquestração** | n8n (Docker) | Workflow automation |
| **IA/LLM** | Google Gemini 2.5 Flash | Interpretação de intenções |
| **Transcrição** | Google Gemini Audio | Speech-to-Text |
| **Banco de Dados** | Supabase (PostgreSQL) | Persistência e histórico |
| **WhatsApp Gateway** | Z-API | Envio/recebimento de mensagens |
| **Calendário** | Google Calendar API | CRUD de eventos |
| **Túnel** | ngrok | Exposição local para webhooks |
| **Container** | Docker + Docker Compose | Infraestrutura |

---

## 📦 Pré-requisitos

Antes de iniciar, você precisará ter:

- Docker e Docker Compose instalados
- Conta no Google Cloud Platform
- Conta no Supabase
- Conta no Z-API (ou similar)
- ngrok instalado (ou domínio público)
- Node.js 18+ (opcional, para desenvolvimento)

---

## 🚀 Setup Completo

### 1. Configuração do Docker (n8n)

Crie o arquivo `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5555:5678"
    environment:
      - N8N_ENCRYPTION_KEY=sua_chave_de_encriptacao_segura
      - POSTGRES_PASSWORD=sua_senha_postgres
      - N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true
      - N8N_EDITOR_BASE_URL=https://seu-dominio.ngrok-free.dev
      - WEBHOOK_URL=https://seu-dominio.ngrok-free.dev
      - N8N_DEFAULT_BINARY_DATA_MODE=filesystem
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
```

Inicie o container:

```bash
docker-compose up -d
```

### 2. Configuração do ngrok

```bash
# Inicie o túnel apontando para a porta do n8n
ngrok http 5555
```

Anote a URL gerada (ex: `https://seu-dominio.ngrok-free.dev`)

### 3. Configuração do Google Cloud Platform

#### 3.1 Criar Projeto

1. Acesse [Google Cloud Console](https://console.cloud.google.com)
2. Crie um novo projeto: "IA para Startup" (ou nome de sua preferência)

#### 3.2 Habilitar APIs

Habilite as seguintes APIs:

- **Google Calendar API**
- **Generative Language API** (Gemini)

#### 3.3 Configurar OAuth 2.0

1. Vá em **APIs e serviços** > **Credenciais**
2. Clique em **Criar credenciais** > **ID do cliente OAuth**
3. Tipo: **Aplicativo da Web**
4. Nome: `n8n oAuth`
5. **Origens JavaScript autorizadas**:
   ```
   https://seu-dominio.ngrok-free.dev
   ```
6. **URIs de redirecionamento autorizados**:
   ```
   https://seu-dominio.ngrok-free.dev/rest/oauth2-credential/callback
   ```
7. Salve o **Client ID** e **Client Secret**

#### 3.4 Configurar Tela de Consentimento

1. Vá em **Google Auth Platform** > **Tela de consentimento do OAuth**
2. Configure como **Externo** (para testes)
3. Adicione os escopos necessários:
   - `https://www.googleapis.com/auth/calendar`
   - `https://www.googleapis.com/auth/calendar.events`

### 4. Configuração do Supabase

#### 4.1 Criar Projeto

1. Acesse [Supabase](https://supabase.com)
2. Crie um novo projeto: `automation-bia`
3. Anote a connection string:
   ```
   postgresql://postgres.[projeto]:[senha]@aws-1-sa-east-1.pooler.supabase.com:5432/postgres
   ```

#### 4.2 Criar Tabelas

Execute os seguintes scripts SQL no SQL Editor:

```sql
-- Tabela de contatos
CREATE TABLE public.contacts (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  phone character varying NOT NULL UNIQUE,
  name character varying,
  created_at timestamp with time zone DEFAULT now(),
  updated_at timestamp with time zone DEFAULT now(),
  CONSTRAINT contacts_pkey PRIMARY KEY (id)
);

-- Tabela de log de mensagens
CREATE TABLE public.messages_log (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  message_id character varying NOT NULL UNIQUE,
  phone character varying NOT NULL,
  direction character varying NOT NULL,
  message_type character varying NOT NULL,
  text text,
  transcript text,
  audio_url text,
  raw_payload jsonb,
  processed boolean DEFAULT false,
  created_at timestamp with time zone DEFAULT now(),
  CONSTRAINT messages_log_pkey PRIMARY KEY (id)
);

-- Tabela de execuções da LLM
CREATE TABLE public.llm_runs (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  message_id character varying,
  model character varying DEFAULT 'gemini-2.5-flash',
  prompt_version character varying DEFAULT 'v1',
  input_text text,
  output_json jsonb,
  success boolean DEFAULT true,
  error_message text,
  retry_count integer DEFAULT 0,
  execution_time_ms integer,
  created_at timestamp with time zone DEFAULT now(),
  CONSTRAINT llm_runs_pkey PRIMARY KEY (id),
  CONSTRAINT llm_runs_message_id_fkey FOREIGN KEY (message_id) 
    REFERENCES public.messages_log(message_id)
);

-- Tabela de erros
CREATE TABLE public.errors (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  source character varying NOT NULL,
  message_id character varying,
  error_code character varying,
  error_message text NOT NULL,
  stack_trace text,
  raw jsonb,
  resolved boolean DEFAULT false,
  created_at timestamp with time zone DEFAULT now(),
  CONSTRAINT errors_pkey PRIMARY KEY (id)
);

-- Tabela de eventos do calendário (opcional - para cache local)
CREATE TABLE public.calendar_events (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  contact_id uuid,
  calendar_event_id character varying UNIQUE,
  title character varying NOT NULL,
  with_person character varying,
  start_at timestamp with time zone NOT NULL,
  end_at timestamp with time zone NOT NULL,
  duration_minutes integer DEFAULT 60,
  notes text,
  status character varying DEFAULT 'active',
  raw_payload jsonb,
  created_at timestamp with time zone DEFAULT now(),
  updated_at timestamp with time zone DEFAULT now(),
  CONSTRAINT calendar_events_pkey PRIMARY KEY (id),
  CONSTRAINT calendar_events_contact_id_fkey FOREIGN KEY (contact_id) 
    REFERENCES public.contacts(id)
);

-- Tabela de histórico de chat do n8n
CREATE TABLE public.n8n_chat_histories (
  id serial PRIMARY KEY,
  session_id character varying NOT NULL,
  message jsonb NOT NULL
);
```

### 5. Configuração do Z-API

1. Acesse [Z-API](https://z-api.io)
2. Crie uma instância
3. Conecte seu WhatsApp (escaneie o QR Code)
4. Configure o webhook de recebimento:
   ```
   https://seu-dominio.ngrok-free.dev/webhook/webhook
   ```
5. Anote as credenciais:
   - **Instance ID**
   - **Token**
   - **Client-Token**

### 6. Configuração das Credenciais no n8n

Acesse o n8n (`https://seu-dominio.ngrok-free.dev`) e configure:

#### 6.1 PostgreSQL (Supabase)

- **Host**: `aws-1-sa-east-1.pooler.supabase.com`
- **Port**: `5432`
- **Database**: `postgres`
- **User**: `postgres.[seu-projeto]`
- **Password**: Sua senha do Supabase

#### 6.2 Google Calendar OAuth2

- **Client ID**: Do Google Cloud
- **Client Secret**: Do Google Cloud
- **Scope**: `https://www.googleapis.com/auth/calendar`

#### 6.3 Google Gemini API

- **API Key**: Gere em Google AI Studio ou Cloud Console

---

## 💾 Schema do Banco de Dados

```
┌─────────────────────┐       ┌─────────────────────┐
│      contacts       │       │    messages_log     │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ phone (UNIQUE)      │       │ message_id (UNIQUE) │
│ name                │       │ phone               │
│ created_at          │       │ direction           │
│ updated_at          │       │ message_type        │
└─────────────────────┘       │ text                │
         │                    │ transcript          │
         │                    │ audio_url           │
         ▼                    │ raw_payload         │
┌─────────────────────┐       │ processed           │
│  calendar_events    │       │ created_at          │
├─────────────────────┤       └──────────┬──────────┘
│ id (PK)             │                  │
│ contact_id (FK)     │                  ▼
│ calendar_event_id   │       ┌─────────────────────┐
│ title               │       │      llm_runs       │
│ with_person         │       ├─────────────────────┤
│ start_at            │       │ id (PK)             │
│ end_at              │       │ message_id (FK)     │
│ duration_minutes    │       │ model               │
│ notes               │       │ prompt_version      │
│ status              │       │ input_text          │
│ raw_payload         │       │ output_json         │
│ created_at          │       │ success             │
│ updated_at          │       │ error_message       │
└─────────────────────┘       │ retry_count         │
                              │ execution_time_ms   │
┌─────────────────────┐       │ created_at          │
│       errors        │       └─────────────────────┘
├─────────────────────┤
│ id (PK)             │       ┌─────────────────────┐
│ source              │       │ n8n_chat_histories  │
│ message_id          │       ├─────────────────────┤
│ error_code          │       │ id (PK)             │
│ error_message       │       │ session_id          │
│ stack_trace         │       │ message (JSONB)     │
│ raw                 │       └─────────────────────┘
│ resolved            │
│ created_at          │
└─────────────────────┘
```

---

## 🔄 Fluxo do Workflow

### Visão Geral dos Nós

```
Webhook → Filter → Check Duplicate → If1 → Main Data → Switch
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
              [is_audio]                [is_text]                [Insert contacts]
                    │                         │
                    ▼                         │
            Parse Audio URL                   │
                    │                         │
                    ▼                         │
            HTTP Request                      │
            (Download)                        │
                    │                         │
                    ▼                         │
            Transcribe                        │
            (Gemini)                          │
                    │                         │
                    ▼                         ▼
            Parse Audio Message         Parse Message
                    │                         │
                    └───────────┬─────────────┘
                                │
                                ▼
                             Merge
                                │
                                ▼
                        Parse Merge Data
                                │
                    ┌───────────┼───────────┐
                    │                       │
                    ▼                       ▼
            Insert messages_log         AI Agent
                                           │
                                           ├───────────────┐
                                           │               │
                                           ▼               ▼
                                    Parse Output AI    [Error Branch]
                                           │               │
                                           ├───────────┐   ▼
                                           │           │  Edit Fields2
                                           ▼           │   │
                                          If           │   ▼
                                           │           │  Insert errors1
                              ┌────────────┼───────┐   │   │
                              │                    │   │   ▼
                              ▼                    ▼   │  Error Friendly
                      [needs_confirm]        [execute] │
                              │                    │   │
                              ▼                    ▼   ▼
                      Confirm Message         Switch1 → Insert llm_runs
                                                  │
                    ┌─────────┬─────────┬─────────┼─────────┬─────────┐
                    │         │         │         │         │         │
                    ▼         ▼         ▼         ▼         ▼         ▼
              out_of_scope  unknown   create   status    update    cancel
                    │         │         │         │         │         │
                    ▼         ▼         ▼         ▼         ▼         ▼
              OFS Message Unknown   Create    Check     Get many   Get many
                          Message   an event  Availability events    events1
                                       │         │         │         │
                                       ▼         ▼         ▼         ▼
                                   Created   Parse     Update    Event Deleted
                                   Event     Response  an event  Message
                                   Message      │         │         │
                                                ▼         ▼         ▼
                                           Message    Event     Delete
                                           Availability Updated  an event
                                                      Message
```

### Descrição Detalhada dos Nós

#### 1. Entrada e Filtros

| Nó | Função |
|----|--------|
| **Webhook** | Recebe POST do Z-API com dados da mensagem |
| **Filter** | Filtra mensagens inválidas (grupos, newsletters, edições, waiting) |
| **Check Duplicate** | Consulta `messages_log` para evitar reprocessamento |
| **If1** | Se duplicado → Do nothing; Senão → continua |

#### 2. Processamento de Dados

| Nó | Função |
|----|--------|
| **Main Data** | Extrai e normaliza dados do webhook (phone, message_id, etc.) |
| **Insert contacts** | Salva/atualiza contato no banco (skip on conflict) |
| **Switch** | Roteia para fluxo de áudio ou texto |

#### 3. Processamento de Áudio

| Nó | Função |
|----|--------|
| **Parse Audio URL** | Extrai URL do áudio |
| **HTTP Request** | Baixa o arquivo de áudio |
| **Transcribe a recording** | Transcreve áudio usando Gemini 2.5 Flash |
| **Parse Audio Message** | Extrai texto da transcrição |

#### 4. Processamento de Texto

| Nó | Função |
|----|--------|
| **Parse Message** | Extrai texto da mensagem |
| **Merge** | Une fluxos de áudio e texto |
| **Parse Merge Data** | Prepara dados para o AI Agent |

#### 5. Inteligência Artificial

| Nó | Função |
|----|--------|
| **AI Agent** | Interpreta intenção usando Gemini com prompt estruturado |
| **Postgres Chat Memory** | Mantém contexto da conversa por telefone |
| **Google Gemini Chat Model** | Modelo de linguagem (temperature: 0.4) |
| **Parse Output AI** | Parseia resposta JSON da IA |
| **Insert llm_runs** | Registra execução da IA no banco |

#### 6. Roteamento de Ações

| Nó | Função |
|----|--------|
| **If** | Verifica se precisa confirmação |
| **Switch1** | Roteia para ação específica (create, update, cancel, status, etc.) |

#### 7. Ações do Google Calendar

| Nó | Função |
|----|--------|
| **Create an event** | Cria novo evento |
| **Update an event** | Atualiza evento existente |
| **Delete an event** | Remove evento |
| **Get availability in a calendar** | Verifica disponibilidade |
| **Get many events** | Lista eventos (para update) |
| **Get many events1** | Lista eventos (para cancel) |

#### 8. Respostas ao Usuário

| Nó | Função |
|----|--------|
| **Confirm Message** | Envia pedido de confirmação |
| **OFS Message** | Resposta para mensagens fora do escopo |
| **Unknown Message** | Resposta para mensagens não compreendidas |
| **Created Event Message** | Confirmação de evento criado |
| **Event Updated Message** | Confirmação de evento atualizado |
| **Event Deleted Message** | Confirmação de evento cancelado |
| **Message Availability** | Resposta sobre disponibilidade |
| **Error Friendly** / **Error Friendly1** | Mensagens amigáveis de erro |

#### 9. Tratamento de Erros

| Nó | Função |
|----|--------|
| **Edit Fields1** / **Edit Fields2** | Formata dados de erro |
| **Insert errors1** / **Insert errors2** | Registra erros no banco |

---

## ✨ Funcionalidades

### Actions Suportadas

| Action | Descrição | Exemplo |
|--------|-----------|---------|
| `create` | Criar novo agendamento | "Marca reunião com João às 14h" |
| `update` | Alterar agendamento existente | "Muda a reunião para 15h" |
| `cancel` | Cancelar agendamento | "Cancela a reunião com Maria" |
| `status` | Consultar disponibilidade | "Estou livre amanhã às 10h?" |
| `deny` | Usuário negou ação proposta | "Não, pode deixar" |
| `unknown` | Intenção não clara | Qualquer mensagem ambígua |
| `out_of_scope` | Fora do tema | "Qual a previsão do tempo?" |

### Regras de Negócio

1. **Horário Comercial**: Segunda a Sexta, 09:00 às 18:00
2. **Duração Padrão**: 60 minutos
3. **Timezone**: America/Sao_Paulo
4. **Confirmação**: Sempre pede confirmação antes de executar ações
5. **Memória**: Mantém contexto da conversa por número de telefone

---

## 📝 Estrutura JSON da IA

O AI Agent retorna um JSON estruturado:

```json
{
  "action": "create|update|cancel|status|deny|unknown|out_of_scope",
  "confidence": 0.95,
  "event": {
    "title": "Reunião com João",
    "with_person": "João",
    "start_datetime": "2026-01-05T14:00:00-03:00",
    "end_datetime": "2026-01-05T15:00:00-03:00",
    "duration_minutes": 60,
    "notes": null
  },
  "target": {
    "calendar_event_id": null,
    "match_strategy": "by_person",
    "person_name": "João"
  },
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": true,
  "user_facing_message": "Vou agendar reunião com João para amanhã às 14h. Confirma?"
}
```

### Match Strategies

| Strategy | Uso | Exemplo |
|----------|-----|---------|
| `by_time` | Encontrar evento por horário | "Muda a reunião das 14h" |
| `by_person` | Encontrar evento por pessoa | "Cancela com o João" |
| `by_title` | Encontrar evento por título | "Cancela a reunião de vendas" |
| `last_confirmed` | Último evento confirmado | "Muda pra amanhã" |

---

## 🔧 Tratamento de Erros

### Estratégias Implementadas

| Estratégia | Implementação |
|------------|---------------|
| **Retry automático** | AI Agent, Google Calendar, HTTP Requests (2 tentativas) |
| **Fallback de erro** | Mensagem amigável ao usuário + registro no banco |
| **Idempotência** | Verificação de `message_id` duplicado |
| **Error Output** | Branch separada para tratamento de erros |

### Mensagem Padrão de Erro

```
Eita! Me enrolei um pouco aqui e não entendi sua mensagem. 🤔 
Consegue repetir pra mim? Se for para agendar, mudar o dia e a hora... 
Se precisar de algo, me avisa!
```

---

## 🧪 Testes Realizados

### Casos de Teste Obrigatórios

| # | Caso de Teste | Status | Observações |
|---|---------------|--------|-------------|
| 1 | Áudio: "Bia, marquei uma reunião às 18:00 com o fulano" | ✅ | Fluxo completo de transcrição + criação |
| 2 | Áudio: "Bia, muda a reunião das 18:00 para 19:00" | ✅ | Update com match_strategy by_time |
| 3 | Áudio: "Bia, cancela a reunião com o fulano" | ✅ | Cancel com match_strategy by_person |
| 4 | Texto: "Bia, status do meu agendamento" | ✅ | Consulta de disponibilidade |
| 5 | Reenvio do mesmo message_id | ✅ | Idempotência funcionando |
| 6 | Forçar falha Gemini | ✅ | Retry + erro amigável + registro no DB |

### Critérios de Avaliação (Rubrica 0-100)

| Critério | Peso | Status |
|----------|------|--------|
| Áudio fim-a-fim (transcrever → interpretar → agir) | 25 | ✅ |
| IA/JSON (schema + guardrails + confirmação) | 20 | ✅ |
| Integrações (Calendar + Z-API + DB) | 20 | ✅ |
| Histórico e rastreabilidade (Postgres) | 15 | ✅ |
| Retry + erro amigável | 10 | ✅ |
| Idempotência + organização/segurança | 10 | ✅ |

---

## 🐛 Troubleshooting

### Problema: Loop infinito de mensagens

**Causa**: A Bia responde e o webhook processa a própria resposta.

**Solução**: Adicionar filtro `fromMe === false` no nó Filter.

```javascript
{{ $json.body.fromMe }} equals false
```

### Problema: Webhook não recebe mensagens

**Causa**: ngrok expirou ou URL mudou.

**Solução**:
1. Reinicie o ngrok
2. Atualize a URL no Z-API
3. Atualize `WEBHOOK_URL` no docker-compose

### Problema: Erro de autenticação Google Calendar

**Causa**: Token OAuth expirado.

**Solução**:
1. Vá em Credenciais no n8n
2. Reconecte a credencial Google Calendar
3. Autorize novamente

### Problema: Mensagens duplicadas sendo processadas

**Causa**: Check Duplicate não está funcionando.

**Solução**: Verifique se a tabela `messages_log` tem índice único em `message_id`.

### Problema: IA retornando update ao invés de create

**Causa**: Prompt não distingue sugestão de evento existente.

**Solução**: O prompt foi atualizado com regras 6, 7 e 8 para distinguir:
- Sugestão (ainda não criado) → `create`
- Evento existente (já criado com ✅) → `update`

---

## 🚀 Melhorias Futuras (Opcionais)

- [ ] Filtro de eventos por `match_strategy` antes de update/cancel
- [ ] Suporte a múltiplos calendários
- [ ] Notificações de lembrete
- [ ] Integração com Google Meet para links automáticos
- [ ] Dashboard de métricas
- [ ] Suporte a recorrência de eventos
- [ ] Cancelamento em lote
- [ ] Integração com outros assistentes (Alexa, Google Assistant)

---

## 📄 Credenciais Necessárias

| Serviço | Credencial | Onde Obter |
|---------|------------|------------|
| **PostgreSQL** | Host, User, Password | Supabase Dashboard |
| **Google Calendar** | OAuth2 (Client ID + Secret) | Google Cloud Console |
| **Google Gemini** | API Key | Google AI Studio |
| **Z-API** | Instance ID, Token, Client-Token | Painel Z-API |

---

## 👤 Autor - Bettin (Jixkls)

Desenvolvido como parte de um teste técnico para demonstrar habilidades em:
- Automação com n8n
- Integração de APIs
- IA Conversacional
- Arquitetura de sistemas

---

## 📜 Licença

Este projeto é apenas para fins de demonstração e avaliação técnica.
