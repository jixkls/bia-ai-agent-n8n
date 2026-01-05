# 🤖 Bia - Assistente Virtual de Agendamentos via WhatsApp

<p align="center">
  <strong>Secretária virtual inteligente que gerencia agendamentos do Google Calendar através do WhatsApp</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow%20Automation-orange" alt="n8n">
  <img src="https://img.shields.io/badge/Google%20Gemini-2.5%20Flash-blue" alt="Gemini">
  <img src="https://img.shields.io/badge/WhatsApp-Z--API-green" alt="WhatsApp">
  <img src="https://img.shields.io/badge/Database-Supabase-dark" alt="Supabase">
  <img src="https://img.shields.io/badge/Calendar-Google%20Calendar-red" alt="Google Calendar">
</p>

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Arquitetura](#-arquitetura)
3. [Stack Tecnológica](#-stack-tecnológica)
4. [Pré-requisitos](#-pré-requisitos)
5. [Setup Completo](#-setup-completo)
6. [Schema do Banco de Dados](#-schema-do-banco-de-dados)
7. [Estrutura do Workflow](#-estrutura-do-workflow)
8. [Prompt do Agente IA](#-prompt-do-agente-ia)
9. [Estrutura JSON da IA](#-estrutura-json-da-ia)
10. [Funcionalidades](#-funcionalidades)
11. [Tratamento de Erros](#-tratamento-de-erros)
12. [Testes Realizados](#-testes-realizados)
13. [Troubleshooting](#-troubleshooting)
14. [Variáveis de Ambiente](#-variáveis-de-ambiente)
15. [Melhorias Futuras](#-melhorias-futuras)

---

## 🎯 Visão Geral

### O que é a Bia?

Bia é um agente conversacional avançado que permite aos usuários gerenciar seus compromissos no Google Calendar através de mensagens no WhatsApp. O sistema processa tanto mensagens de texto quanto áudios, interpretando a intenção do usuário em linguagem natural e executando ações no calendário de forma inteligente.

### Casos de Uso Principais

| Ação | Exemplo de Mensagem | Resposta da Bia |
|------|---------------------|-----------------|
| **Criar agendamento** | "Bia, marca uma reunião com o João amanhã às 14h" | "Vou verificar a disponibilidade e agendar reunião com João amanhã às 14h. Confirma?" |
| **Alterar evento** | "Bia, muda a reunião das 14h para 15h" | "O horário do agendamento foi alterado para o dia XX/XX/XXXX das 15:00 às 16:00..." |
| **Cancelar evento** | "Bia, cancela a reunião com a Maria" | "Compromisso cancelado!! Quer que eu faça mais algo?" |
| **Consultar agenda** | "Bia, estou livre amanhã às 10h?" | "✅ Ótima notícia! O horário está livre! Deseja que eu agende algo?" |
| **Negar ação** | "Não, pode deixar" | "Ok, sem problemas! Se precisar, é só chamar 😊" |

### Diferenciais Técnicos

- ✅ **Suporte completo a áudio** - Transcrição automática via Google Gemini 2.5 Flash
- ✅ **Memória de contexto** - Histórico de conversas por telefone via PostgreSQL
- ✅ **Validação de horário comercial** - Segunda a Sexta, 09:00-18:00
- ✅ **Idempotência garantida** - Prevenção de processamento duplicado via `message_id`
- ✅ **Tratamento robusto de erros** - Mensagens amigáveis + registro completo no banco
- ✅ **Logs completos** - Rastreabilidade total de todas as operações
- ✅ **Retry automático** - Tentativas com backoff progressivo em falhas
- ✅ **Verificação de disponibilidade** - Consulta FreeBusy antes de criar eventos

---

## 🏗 Arquitetura

### Diagrama de Arquitetura

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    ENTRADA                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐    │
│  │  WhatsApp   │────▶│   Z-API     │────▶│   ngrok     │────▶│  n8n Webhook    │    │
│  │  (Usuário)  │     │  (Gateway)  │     │  (Túnel)    │     │  (Entrada)      │    │
│  └─────────────┘     └─────────────┘     └─────────────┘     └────────┬────────┘    │
│                                                                        │             │
└────────────────────────────────────────────────────────────────────────┼─────────────┘
                                                                         │
┌────────────────────────────────────────────────────────────────────────┼─────────────┐
│                              PROCESSAMENTO                              │             │
├────────────────────────────────────────────────────────────────────────┼─────────────┤
│                                                                        ▼             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           n8n Workflow Engine                                │    │
│  │  ┌───────────────┐   ┌───────────────┐   ┌───────────────────────────────┐  │    │
│  │  │    Filter     │──▶│   Check       │──▶│     Main Data Parser          │  │    │
│  │  │ (Validação)   │   │   Duplicate   │   │  (Normalização de Dados)      │  │    │
│  │  └───────────────┘   └───────────────┘   └───────────────┬───────────────┘  │    │
│  │                                                          │                   │    │
│  │                              ┌───────────────────────────┴───────────┐       │    │
│  │                              ▼                                       ▼       │    │
│  │                    ┌─────────────────┐                    ┌─────────────────┐│    │
│  │                    │   Audio Flow    │                    │   Text Flow     ││    │
│  │                    │  ┌───────────┐  │                    │  ┌───────────┐  ││    │
│  │                    │  │HTTP Request│  │                    │  │  Parse    │  ││    │
│  │                    │  │(Download) │  │                    │  │  Message  │  ││    │
│  │                    │  └─────┬─────┘  │                    │  └─────┬─────┘  ││    │
│  │                    │        ▼        │                    │        │        ││    │
│  │                    │  ┌───────────┐  │                    │        │        ││    │
│  │                    │  │ Gemini    │  │                    │        │        ││    │
│  │                    │  │Transcribe │  │                    │        │        ││    │
│  │                    │  └─────┬─────┘  │                    │        │        ││    │
│  │                    └────────┼────────┘                    └────────┼────────┘│    │
│  │                             │                                      │         │    │
│  │                             └──────────────┬────────────────────────┘         │    │
│  │                                            ▼                                  │    │
│  │                              ┌─────────────────────────┐                      │    │
│  │                              │        Merge Node       │                      │    │
│  │                              │   (Unifica Fluxos)      │                      │    │
│  │                              └────────────┬────────────┘                      │    │
│  │                                           ▼                                   │    │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │    │
│  │  │                         AI Agent (Gemini 2.5 Flash)                     │  │    │
│  │  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │  │    │
│  │  │  │ Postgres Chat    │  │ Google Gemini    │  │   Structured JSON    │  │  │    │
│  │  │  │ Memory           │  │ Chat Model       │  │   Output Parser      │  │  │    │
│  │  │  │ (Contexto)       │  │ (temp: 0.4)      │  │   (Action Router)    │  │  │    │
│  │  │  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │  │    │
│  │  └────────────────────────────────────────────────────────────────────────┘  │    │
│  │                                           │                                   │    │
│  │                                           ▼                                   │    │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │    │
│  │  │                         Action Router (Switch)                          │  │    │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │  │    │
│  │  │  │  CREATE  │ │  UPDATE  │ │  CANCEL  │ │  STATUS  │ │ OUT_OF_SCOPE │  │  │    │
│  │  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘  │  │    │
│  │  └───────┼────────────┼───────────┼────────────┼───────────────┼──────────┘  │    │
│  └──────────┼────────────┼───────────┼────────────┼───────────────┼─────────────┘    │
└─────────────┼────────────┼───────────┼────────────┼───────────────┼──────────────────┘
              │            │           │            │               │
┌─────────────┼────────────┼───────────┼────────────┼───────────────┼──────────────────┐
│             ▼            ▼           ▼            ▼               ▼       INTEGRAÇÕES│
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                              Google Calendar API                                 ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  ││
│  │  │  FreeBusy   │  │   Create    │  │   Update    │  │       Delete          │  ││
│  │  │  (Check)    │  │   Event     │  │   Event     │  │       Event           │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                           Supabase (PostgreSQL)                                  ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  ││
│  │  │  contacts   │  │messages_log │  │  llm_runs   │  │   calendar_events     │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────────────────┘  ││
│  │  ┌─────────────┐  ┌─────────────────────────────────────────────────────────┐  ││
│  │  │   errors    │  │              n8n_chat_histories                          │  ││
│  │  └─────────────┘  └─────────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                      SAÍDA                                           │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────────────┐│
│  │  n8n HTTP       │────▶│     Z-API       │────▶│           WhatsApp              ││
│  │  Request        │     │   Send Text     │     │         (Usuário)               ││
│  │  (Resposta)     │     │                 │     │                                 ││
│  └─────────────────┘     └─────────────────┘     └─────────────────────────────────┘│
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### Fluxo de Dados Simplificado

```
1. Usuário envia mensagem (texto/áudio) ──▶ WhatsApp
                                              │
2. Z-API captura mensagem ◀──────────────────┘
                │
3. Webhook ngrok recebe payload ◀────────────┘
                │
4. n8n processa:
   ├── Filtra mensagens válidas (não fromMe, não grupo, não editado)
   ├── Verifica idempotência (message_id único)
   ├── Se áudio: baixa mídia → transcreve com Gemini
   ├── AI Agent interpreta intenção
   ├── Valida JSON de resposta
   └── Roteia para ação apropriada
                │
5. Executa ação no Google Calendar ◀─────────┘
                │
6. Persiste dados no Supabase ◀──────────────┘
                │
7. Envia resposta via Z-API ──▶ WhatsApp ──▶ Usuário
```

---

## 🛠 Stack Tecnológica

| Componente | Tecnologia | Versão | Função |
|------------|------------|--------|--------|
| **Orquestração** | n8n | latest | Workflow automation engine |
| **IA/LLM** | Google Gemini | 2.5 Flash | Interpretação de intenções e geração de respostas |
| **Transcrição** | Google Gemini Audio | 2.5 Flash | Speech-to-Text para mensagens de áudio |
| **Banco de Dados** | Supabase | PostgreSQL | Persistência, histórico e memória de chat |
| **WhatsApp Gateway** | Z-API | - | Envio e recebimento de mensagens |
| **Calendário** | Google Calendar API | v3 | CRUD de eventos e verificação de disponibilidade |
| **Túnel** | ngrok | 3.34.1 | Exposição local para webhooks |
| **Container** | Docker + Docker Compose | latest | Infraestrutura containerizada |
| **Região** | South America (sa) | - | Baixa latência (~15ms) |

---

## 📦 Pré-requisitos

### Software Necessário

| Requisito | Versão Mínima | Verificação |
|-----------|---------------|-------------|
| Docker | 20.10+ | `docker --version` |
| Docker Compose | 2.0+ | `docker compose version` |
| ngrok | 3.x | `ngrok --version` |
| Node.js | 18+ | `node --version` (opcional) |

### Contas e Serviços

- [ ] **Google Cloud Platform** - Projeto com APIs habilitadas
- [ ] **Supabase** - Projeto com banco PostgreSQL
- [ ] **Z-API** - Instância ativa com WhatsApp conectado
- [ ] **ngrok** - Conta (gratuita funciona)

---

## 🚀 Setup Completo

### 1. Configuração do Docker (n8n)

#### 1.1 Criar arquivo `docker-compose.yml`

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5555:5678"
    environment:
      - N8N_ENCRYPTION_KEY=sua_chave_de_encriptacao_segura_32_chars
      - POSTGRES_PASSWORD=sua_senha_postgres_segura
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

#### 1.2 Iniciar o container

```bash
# Iniciar em background
docker-compose up -d

# Verificar status
docker stats

# Ver logs
docker logs -f n8n-n8n-1
```

### 2. Configuração do ngrok

#### 2.1 Autenticar ngrok

```bash
ngrok config add-authtoken SEU_AUTH_TOKEN
```

#### 2.2 Iniciar túnel

```bash
# Apontar para a porta do n8n
ngrok http 5555
```

#### 2.3 Verificar conexão

Após iniciar, você verá:

```
Session Status                online
Account                       SeuUsuario (Plan: Free)
Version                       3.34.1
Region                        South America (sa)
Latency                       15ms
Web Interface                 http://127.0.0.1:4040
Forwarding                    https://stagnatory-unaustere-darien.ngrok-free.dev -> http://localhost:5555
```

> ⚠️ **Importante**: Anote a URL de Forwarding (ex: `https://stagnatory-unaustere-darien.ngrok-free.dev`)

### 3. Configuração do Google Cloud Platform

#### 3.1 Criar Projeto

1. Acesse [Google Cloud Console](https://console.cloud.google.com)
2. Clique em **Selecionar projeto** → **Novo projeto**
3. Nome: `IA para Startup` (ou sua preferência)
4. Clique em **Criar**

#### 3.2 Habilitar APIs

No menu lateral, vá em **APIs e serviços** → **Biblioteca** e habilite:

| API | Função |
|-----|--------|
| **Google Calendar API** | CRUD de eventos, FreeBusy |
| **Generative Language API** | Gemini 2.5 Flash |

#### 3.3 Configurar OAuth 2.0

1. Vá em **APIs e serviços** → **Credenciais**
2. Clique em **Criar credenciais** → **ID do cliente OAuth**
3. Configure:

| Campo | Valor |
|-------|-------|
| Tipo de aplicativo | Aplicativo da Web |
| Nome | `n8n oAuth` |
| Origens JavaScript autorizadas | `https://seu-dominio.ngrok-free.dev` |
| URIs de redirecionamento autorizados | `https://seu-dominio.ngrok-free.dev/rest/oauth2-credential/callback` |

4. Clique em **Criar**
5. **Salve** o `Client ID` e `Client Secret`

#### 3.4 Configurar Tela de Consentimento

1. Vá em **Google Auth Platform** → **Tela de consentimento do OAuth**
2. Tipo de usuário: **Externo**
3. Adicione escopos:
   - `https://www.googleapis.com/auth/calendar`
   - `https://www.googleapis.com/auth/calendar.events`
4. Adicione usuários de teste (seu e-mail)

#### 3.5 Obter API Key do Gemini

1. Acesse [Google AI Studio](https://aistudio.google.com)
2. Clique em **Get API Key**
3. Crie uma nova chave ou use existente
4. **Salve** a API Key

### 4. Configuração do Supabase

#### 4.1 Criar Projeto

1. Acesse [Supabase](https://supabase.com)
2. Clique em **New Project**
3. Configure:

| Campo | Valor |
|-------|-------|
| Nome | `automation-bia` |
| Database Password | Senha forte |
| Region | South America (São Paulo) |

4. **Salve** a senha do banco

#### 4.2 Obter Connection String

Em **Settings** → **Database**, copie a connection string:

```
postgresql://postgres.[projeto]:[senha]@aws-1-sa-east-1.pooler.supabase.com:5432/postgres
```

#### 4.3 Criar Tabelas

Execute no **SQL Editor** do Supabase:

```sql
-- Ver seção "Schema do Banco de Dados" abaixo
```

### 5. Configuração do Z-API

#### 5.1 Criar Instância

1. Acesse o [Painel Z-API](https://app.z-api.io)
2. Crie uma nova instância
3. Conecte seu WhatsApp via QR Code

#### 5.2 Configurar Webhook

1. Em **Configurações** → **Webhooks**
2. Configure:

| Campo | Valor |
|-------|-------|
| URL de Recebimento | `https://seu-dominio.ngrok-free.dev/webhook/webhook-message` |
| Eventos | `Mensagens recebidas` |

#### 5.3 Obter Credenciais

Anote:
- **Instance ID**: `3ECC141C33A6E1DC10AFDAD6DC7CFA85`
- **Token**: `803289FE9E41E2587BE005EA`
- **Client-Token**: `Fc100e77c09864bd09c30e5537eb35766S`

### 6. Configurar Credenciais no n8n

1. Acesse `https://seu-dominio.ngrok-free.dev`
2. Vá em **Settings** → **Credentials**
3. Adicione:

| Credencial | Tipo | Campos |
|------------|------|--------|
| PostgreSQL | Postgres | Host, Database, User, Password |
| Google Calendar | OAuth2 | Client ID, Client Secret |
| Google Gemini | API Key | API Key |

### 7. Importar Workflow

1. Vá em **Workflows** → **Import from File**
2. Selecione o arquivo `workflow.json`
3. Ative o workflow

---

## 🗄 Schema do Banco de Dados

### Diagrama ER

```
┌─────────────────────┐       ┌─────────────────────────┐
│      contacts       │       │      messages_log       │
├─────────────────────┤       ├─────────────────────────┤
│ id (PK, uuid)       │       │ id (PK, uuid)           │
│ phone (UNIQUE)      │◀──────│ phone                   │
│ name                │       │ message_id (UNIQUE)     │
│ created_at          │       │ direction               │
│ updated_at          │       │ message_type            │
└─────────────────────┘       │ text                    │
         │                    │ transcript              │
         │                    │ audio_url               │
         │                    │ raw_payload (JSONB)     │
         │                    │ processed               │
         │                    │ created_at              │
         │                    └───────────┬─────────────┘
         │                                │
         ▼                                ▼
┌─────────────────────┐       ┌─────────────────────────┐
│  calendar_events    │       │       llm_runs          │
├─────────────────────┤       ├─────────────────────────┤
│ id (PK, uuid)       │       │ id (PK, uuid)           │
│ contact_id (FK)     │       │ message_id (FK)         │
│ calendar_event_id   │       │ model                   │
│ title               │       │ prompt_version          │
│ with_person         │       │ input_text              │
│ start_at            │       │ output_json (JSONB)     │
│ end_at              │       │ success                 │
│ duration_minutes    │       │ error_message           │
│ notes               │       │ retry_count             │
│ status              │       │ execution_time_ms       │
│ raw_payload (JSONB) │       │ created_at              │
│ created_at          │       └─────────────────────────┘
│ updated_at          │
└─────────────────────┘
                              ┌─────────────────────────┐
┌─────────────────────┐       │   n8n_chat_histories    │
│       errors        │       ├─────────────────────────┤
├─────────────────────┤       │ id (PK, serial)         │
│ id (PK, uuid)       │       │ session_id              │
│ source              │       │ message (JSONB)         │
│ message_id          │       └─────────────────────────┘
│ error_code          │
│ error_message       │
│ stack_trace         │
│ raw (JSONB)         │
│ resolved            │
│ created_at          │
└─────────────────────┘
```

### Scripts SQL

```sql
-- Extensão para UUID
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

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

-- Tabela de eventos do calendário
CREATE TABLE public.calendar_events (
  id uuid NOT NULL DEFAULT uuid_generate_v4(),
  contact_id uuid,
  calendar_event_id character varying NOT NULL UNIQUE,
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

-- Tabela de histórico de chat do n8n
CREATE TABLE public.n8n_chat_histories (
  id serial NOT NULL,
  session_id character varying NOT NULL,
  message jsonb NOT NULL,
  CONSTRAINT n8n_chat_histories_pkey PRIMARY KEY (id)
);

-- Índices para performance
CREATE INDEX idx_messages_log_phone ON public.messages_log(phone);
CREATE INDEX idx_messages_log_created_at ON public.messages_log(created_at);
CREATE INDEX idx_calendar_events_start_at ON public.calendar_events(start_at);
CREATE INDEX idx_errors_created_at ON public.errors(created_at);
CREATE INDEX idx_n8n_chat_histories_session_id ON public.n8n_chat_histories(session_id);
```

---

## 🔄 Estrutura do Workflow

### Visão Geral dos Nós (79 nós totais)

O workflow está organizado em 9 seções principais:

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. ENTRADA E FILTROS (5 nós)                                       │
│     Webhook → Filter → Check Duplicate → If1 → Do nothing           │
├─────────────────────────────────────────────────────────────────────┤
│  2. NORMALIZAÇÃO DE DADOS (2 nós)                                   │
│     Main Data → Insert contacts                                      │
├─────────────────────────────────────────────────────────────────────┤
│  3. PROCESSAMENTO DE ÁUDIO (5 nós)                                  │
│     Switch → Parse Audio URL → HTTP Request → Transcribe → Parse    │
├─────────────────────────────────────────────────────────────────────┤
│  4. PROCESSAMENTO DE TEXTO (1 nó)                                   │
│     Parse Message                                                    │
├─────────────────────────────────────────────────────────────────────┤
│  5. MERGE E IA (7 nós)                                              │
│     Merge → Parse Merge Data → Insert messages_log →                │
│     AI Agent + Gemini Model + Postgres Memory → Parse Output AI      │
├─────────────────────────────────────────────────────────────────────┤
│  6. ROTEAMENTO E CONFIRMAÇÃO (4 nós)                                │
│     Insert llm_runs → If → Confirm Message → Switch1                 │
├─────────────────────────────────────────────────────────────────────┤
│  7. AÇÕES DO GOOGLE CALENDAR (25 nós)                               │
│     CREATE: Check Availability → Create Event → Insert DB → Message │
│     UPDATE: Get Events → Select DB → Update → Insert DB → Message   │
│     CANCEL: Get Events → Delete Event → Delete DB → Message         │
│     STATUS: Get Availability → If → Get Events → Format → Message   │
├─────────────────────────────────────────────────────────────────────┤
│  8. RESPOSTAS AO USUÁRIO (12 nós)                                   │
│     OFS Message, Unknown Message, Created Event Message,            │
│     Event Updated Message, Event Deleted Message,                    │
│     Check Availability Messages, Not Found Event Message            │
├─────────────────────────────────────────────────────────────────────┤
│  9. TRATAMENTO DE ERROS (18 nós)                                    │
│     Error Parser (x11) → Insert errors (x11) → Error Friendly (x11) │
└─────────────────────────────────────────────────────────────────────┘
```

### Detalhamento por Seção

#### 1. Entrada e Filtros

| Nó | Tipo | Função | Configuração |
|----|------|--------|--------------|
| **Webhook** | `n8n-nodes-base.webhook` | Recebe POST do Z-API | Path: `/webhook-message`, Method: POST |
| **Filter** | `n8n-nodes-base.filter` | Valida mensagens | `waitingMessage=false`, `isEdit=false`, `isGroup=false`, `isNewsletter=false`, `fromMe=false` |
| **Check Duplicate** | `n8n-nodes-base.postgres` | Verifica idempotência | `SELECT EXISTS(SELECT 1 FROM messages_log WHERE message_id = ?)` |
| **If1** | `n8n-nodes-base.if` | Roteia duplicados | Se `is_duplicate=true` → Do nothing |
| **Do nothing** | `n8n-nodes-base.noOp` | Ignora duplicados | - |

#### 2. Normalização de Dados

| Nó | Tipo | Campos Extraídos |
|----|------|------------------|
| **Main Data** | `n8n-nodes-base.set` | `message_id`, `phone`, `from_me`, `timestamp`, `instance_id`, `chat_name`, `sender_name`, `message_type`, `text_message`, `audio_url`, `audio_duration`, `audio_ptt`, `is_audio`, `is_text`, `direction`, `current_datetime`, `current_datetime_sao_paulo`, `workflow_execution_id` |
| **Insert contacts** | `n8n-nodes-base.postgres` | Insere/atualiza contato com `skipOnConflict: true` |

#### 3. Processamento de Áudio

| Nó | Tipo | Função | Retry |
|----|------|--------|-------|
| **Switch** | `n8n-nodes-base.switch` | Roteia audio vs texto | - |
| **Parse Audio URL** | `n8n-nodes-base.set` | Extrai URL do áudio | - |
| **HTTP Request** | `n8n-nodes-base.httpRequest` | Baixa arquivo de áudio | - |
| **Transcribe a recording** | `@n8n/n8n-nodes-langchain.googleGemini` | Transcreve com Gemini 2.5 Flash | 2 tentativas |
| **Parse Audio Message** | `n8n-nodes-base.set` | Extrai texto da transcrição | - |

#### 4. Processamento de Texto

| Nó | Tipo | Função |
|----|------|--------|
| **Parse Message** | `n8n-nodes-base.set` | Extrai `text_message` |

#### 5. Merge e IA

| Nó | Tipo | Função | Configuração |
|----|------|--------|--------------|
| **Merge** | `n8n-nodes-base.merge` | Une fluxos audio/texto | - |
| **Parse Merge Data** | `n8n-nodes-base.set` | Prepara dados para IA | - |
| **Insert messages_log** | `n8n-nodes-base.postgres` | Persiste mensagem | 3 tentativas |
| **AI Agent** | `@n8n/n8n-nodes-langchain.agent` | Interpreta intenção | 2 tentativas |
| **Google Gemini Chat Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Modelo LLM | `temperature: 0.4` |
| **Postgres Chat Memory** | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Contexto por telefone | `sessionKey: phone` |
| **Parse Output AI** | `n8n-nodes-base.set` | Parseia JSON da resposta | - |

#### 6. Roteamento e Confirmação

| Nó | Tipo | Função |
|----|------|--------|
| **Insert llm_runs** | `n8n-nodes-base.postgres` | Registra execução LLM |
| **If** | `n8n-nodes-base.if` | Verifica `needs_confirmation` ou datas nulas |
| **Confirm Message** | `n8n-nodes-base.httpRequest` | Envia confirmação via Z-API |
| **Switch1** | `n8n-nodes-base.switch` | Roteia por `action` |

#### 7. Ações do Google Calendar

##### CREATE Flow

| Nó | Função |
|----|--------|
| **Check Avaliability2** | Envia mensagem de confirmação |
| **Get availability in a calendar1** | Verifica FreeBusy |
| **If2** | Se disponível → cria, senão → informa conflito |
| **Create an event** | Cria evento no Google Calendar |
| **Edit Fields** | Formata dados para persistência |
| **Insert Calendar Events** | Persiste no Supabase |
| **Created Event Message** | Envia confirmação ao usuário |
| **Check Avaliability1** | Informa conflito se indisponível |

##### UPDATE Flow

| Nó | Função |
|----|--------|
| **Get many events** | Lista eventos do dia |
| **Select Calendar Events** | Busca evento no banco |
| **Event Updated Message1** | Confirma ação |
| **If3** | Verifica se evento existe |
| **Update an event** | Atualiza no Google Calendar |
| **Edit Fields1** | Formata dados atualizados |
| **Insert Calendar Events1** | Atualiza no Supabase (upsert) |
| **Event Updated Message** | Confirma atualização |
| **Not Found Event Message** | Informa se não encontrado |

##### CANCEL Flow

| Nó | Função |
|----|--------|
| **Get many events2** | Lista eventos do dia |
| **Event Deleted Message** | Confirma cancelamento |
| **Delete an event** | Remove do Google Calendar |
| **Delete table or rows** | Remove do Supabase |
| **Event Deleted Message1** | Confirmação final |

##### STATUS Flow

| Nó | Função |
|----|--------|
| **Check Avaliability** | Envia mensagem inicial |
| **Get availability in a calendar** | Verifica FreeBusy |
| **If5** | Roteia baseado em disponibilidade |
| **Check Avaliability3** | Informa se livre |
| **Get many events1** | Lista eventos se ocupado |
| **Edit Fields2** | Formata dados dos eventos |
| **Message Avaliability1** | Envia detalhes dos compromissos |

#### 8. Respostas ao Usuário

| Nó | Trigger | Mensagem |
|----|---------|----------|
| **OFS Message** | `action=out_of_scope` | `user_facing_message` da IA |
| **Unknown Message** | `action=unknown` | Mensagem de erro amigável |
| **Confirm Message** | `needs_confirmation=true` | `user_facing_message` da IA |
| **Created Event Message** | Evento criado | "Seu horário foi agendado com sucesso..." |
| **Event Updated Message** | Evento atualizado | "O horário do agendamento foi alterado..." |
| **Event Deleted Message1** | Evento cancelado | "Compromisso cancelado!!" |
| **Message Avaliability1** | Status com eventos | Detalhes formatados dos compromissos |

#### 9. Tratamento de Erros

Cada seção crítica possui um par de nós de erro:

| Error Parser | Insert errors | Error Friendly | Origem |
|--------------|---------------|----------------|--------|
| Error Parser | Insert errors | Error Firendly2 | Parse Output AI |
| Error Parser1 | Insert errors1 | Error Firendly | AI Agent |
| Error Parser2 | Insert errors2 | Error Firendly1 | Ações Calendar/Messages |
| Error Parser3 | Insert errors3 | Error Firendly3 | Main Data |
| Error Parser4 | Insert errors4 | Error Firendly4 | Transcribe |
| Error Parser8 | Insert errors8 | Error Firendly8 | Insert Calendar Events |
| Error Parser9 | Insert errors9 | Error Firendly9 | Insert Calendar Events1 |
| Error Parser10 | Insert errors10 | Error Firendly10 | Delete table or rows |

---

## 🤖 Prompt do Agente IA

### Prompt Completo

```
Você é Bia, secretária virtual do WhatsApp que gerencia Google Calendar.

CONTEXTO:
- Timezone: America/Sao_Paulo | Agora: {{$now.toISO()}}
- Horário comercial: Seg-Sex, 09:00-18:00 | Duração padrão: 60min
- Usuário: {{ $json.sender_name }} ({{ $json.phone }})
- Mensagem: {{ $json.message }}

ACTIONS:
- create: agendar evento
- update: alterar evento
- cancel: cancelar evento
- status: consultar agenda
- deny: usuário negou ação proposta
- unknown: intenção não clara (needs_clarification=true)
- out_of_scope: 100% fora do tema agendamentos

REGRAS CRÍTICAS:
1. Saudação + pedido = FOQUE NO PEDIDO (não é out_of_scope)
2. Confirmação ("sim", "ok", "confirma") = use ACTION ORIGINAL com needs_confirmation:false
3. Horário sem data: se não passou hoje, use hoje; senão amanhã
4. Fora do horário comercial/fim de semana: sugira alternativa
5. STATUS: end_datetime = start + 60min (dia inteiro: 09:00-18:00)
6. IMPORTANTE: "update" SOMENTE para eventos JÁ CRIADOS no Google Calendar. 
   Se o usuário está corrigindo/aceitando uma SUGESTÃO sua antes de criar, 
   ainda é "create" (o evento não existe ainda).
7. Se needs_confirmation foi true na resposta anterior e usuário confirma/ajusta, 
   mantenha action original (create→create, update→update, cancel→cancel).
8. COMO SABER SE EVENTO EXISTE:
   - Se você acabou de sugerir criar algo = NÃO EXISTE ainda
   - Se o usuário menciona algo que você criou antes (com ✅) = EXISTE
   - Na dúvida, pergunte: "Você quer que eu crie um novo agendamento ou alterar um existente?"

VERIFICAÇÃO DE DISPONIBILIDADE:
9. NUNCA assuma que um horário está livre. O sistema verificará automaticamente.
10. Ao criar evento, SEMPRE use needs_confirmation:true primeiro para confirmar com usuário.
11. Na user_facing_message de confirmação, diga que vai "verificar a disponibilidade e agendar".
12. NÃO diga "Reunião agendada ✅" até que o usuário confirme E o sistema valide o horário.
13. Se o usuário confirmar e houver conflito, o sistema informará automaticamente.

MATCH STRATEGIES (update/cancel): by_time | by_person | by_title | last_confirmed

FORMATO DE RESPOSTA (OBRIGATÓRIO):
Responda EXCLUSIVAMENTE com um objeto JSON válido.

PROIBIDO:
❌ ```json
❌ ```
❌ Qualquer texto antes do {
❌ Qualquer texto depois do }
❌ Explicações ou comentários

OBRIGATÓRIO:
✅ Primeiro caractere da resposta: {
✅ Último caractere da resposta: }
```

### Regras de Negócio Implementadas

| Regra | Descrição | Implementação |
|-------|-----------|---------------|
| **Timezone** | America/Sao_Paulo | Todas as datas em `-03:00` |
| **Horário Comercial** | Seg-Sex, 09:00-18:00 | IA sugere alternativas fora |
| **Duração Padrão** | 60 minutos | `duration_minutes: 60` |
| **Confirmação** | Sempre antes de ações | `needs_confirmation: true` |
| **Memória** | Contexto por telefone | Postgres Chat Memory |
| **Idempotência** | `message_id` único | Check Duplicate query |

---

## 📝 Estrutura JSON da IA

### Schema Completo (v1)

```json
{
  "action": "create|update|cancel|status|deny|unknown|out_of_scope",
  "confidence": 0.0-1.0,
  "event": {
    "title": "string|null",
    "with_person": "string|null",
    "start_datetime": "YYYY-MM-DDTHH:MM:SS-03:00|null",
    "end_datetime": "YYYY-MM-DDTHH:MM:SS-03:00|null",
    "duration_minutes": 60,
    "notes": "string|null"
  },
  "target": {
    "calendar_event_id": "string|null",
    "match_strategy": "by_time|by_person|by_title|last_confirmed|null",
    "person_name": "string|null"
  },
  "needs_clarification": false,
  "clarification_question": "string|null",
  "needs_confirmation": true|false,
  "user_facing_message": "string"
}
```

### Exemplos por Action

#### CREATE (pedindo confirmação)

```json
{
  "action": "create",
  "confidence": 0.95,
  "event": {
    "title": "Reunião com João",
    "with_person": "João",
    "start_datetime": "2026-01-06T14:00:00-03:00",
    "end_datetime": "2026-01-06T15:00:00-03:00",
    "duration_minutes": 60,
    "notes": null
  },
  "target": null,
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": true,
  "user_facing_message": "Vou verificar a disponibilidade e agendar reunião com João amanhã às 14h. Confirma?"
}
```

#### CREATE (após confirmação)

```json
{
  "action": "create",
  "confidence": 0.98,
  "event": {
    "title": "Reunião com João",
    "with_person": "João",
    "start_datetime": "2026-01-06T14:00:00-03:00",
    "end_datetime": "2026-01-06T15:00:00-03:00",
    "duration_minutes": 60,
    "notes": null
  },
  "target": null,
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": false,
  "user_facing_message": "Verificando disponibilidade e agendando..."
}
```

#### UPDATE

```json
{
  "action": "update",
  "confidence": 0.90,
  "event": {
    "title": "Reunião com João",
    "with_person": "João",
    "start_datetime": "2026-01-06T15:00:00-03:00",
    "end_datetime": "2026-01-06T16:00:00-03:00",
    "duration_minutes": 60,
    "notes": null
  },
  "target": {
    "calendar_event_id": null,
    "match_strategy": "by_time",
    "person_name": "João"
  },
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": true,
  "user_facing_message": "Vou alterar a reunião para 15h. Confirma?"
}
```

#### CANCEL

```json
{
  "action": "cancel",
  "confidence": 0.90,
  "event": null,
  "target": {
    "calendar_event_id": null,
    "match_strategy": "by_person",
    "person_name": "Maria"
  },
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": true,
  "user_facing_message": "Vou cancelar a reunião com Maria. Confirma?"
}
```

#### STATUS

```json
{
  "action": "status",
  "confidence": 0.95,
  "event": {
    "title": null,
    "with_person": null,
    "start_datetime": "2026-01-10T10:00:00-03:00",
    "end_datetime": "2026-01-10T11:00:00-03:00",
    "duration_minutes": 60,
    "notes": null
  },
  "target": {
    "calendar_event_id": null,
    "match_strategy": "by_time"
  },
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": false,
  "user_facing_message": "Vou verificar sua agenda para o dia 10 às 10h!"
}
```

#### DENY

```json
{
  "action": "deny",
  "confidence": 0.95,
  "event": null,
  "target": null,
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": false,
  "user_facing_message": "Ok, sem problemas! Se precisar, é só chamar 😊"
}
```

#### OUT_OF_SCOPE

```json
{
  "action": "out_of_scope",
  "confidence": 1.0,
  "event": null,
  "target": null,
  "needs_clarification": false,
  "clarification_question": null,
  "needs_confirmation": false,
  "user_facing_message": "Sou especializada em agendamentos! Posso marcar algo pra você?"
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

## ✨ Funcionalidades

### Actions Suportadas

| Action | Descrição | Trigger | Requer Confirmação |
|--------|-----------|---------|-------------------|
| `create` | Criar novo agendamento | "Marca reunião com X às Yh" | ✅ Sim |
| `update` | Alterar agendamento existente | "Muda a reunião para Xh" | ✅ Sim |
| `cancel` | Cancelar agendamento | "Cancela a reunião com X" | ✅ Sim |
| `status` | Consultar disponibilidade | "Estou livre amanhã às Xh?" | ❌ Não |
| `deny` | Usuário negou ação proposta | "Não", "Deixa pra lá" | ❌ Não |
| `unknown` | Intenção não clara | Mensagens ambíguas | ❌ Não |
| `out_of_scope` | Fora do tema | "Qual a previsão do tempo?" | ❌ Não |

### Fluxo de Confirmação

```
┌─────────────────────────────────────────────────────────────────┐
│  Usuário: "Marca reunião com João às 14h"                       │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Bia: "Vou verificar a disponibilidade e agendar reunião com    │
│        João amanhã às 14h. Confirma?"                           │
│  [needs_confirmation: true]                                      │
└─────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
┌─────────────────────────┐      ┌─────────────────────────────────┐
│  Usuário: "Sim"         │      │  Usuário: "Não"                 │
│  [confirma]             │      │  [nega]                         │
└───────────┬─────────────┘      └───────────────┬─────────────────┘
            │                                    │
            ▼                                    ▼
┌─────────────────────────┐      ┌─────────────────────────────────┐
│  Sistema verifica       │      │  Bia: "Ok, sem problemas!       │
│  disponibilidade via    │      │        Se precisar, é só        │
│  FreeBusy               │      │        chamar 😊"               │
└───────────┬─────────────┘      │  [action: deny]                 │
            │                    └─────────────────────────────────┘
   ┌────────┴────────┐
   ▼                 ▼
┌──────────┐    ┌──────────────────────────────────────────────────┐
│ Livre    │    │  Ocupado                                         │
└────┬─────┘    └───────────────────────┬──────────────────────────┘
     │                                  │
     ▼                                  ▼
┌──────────────────────────┐   ┌──────────────────────────────────┐
│  Cria evento no          │   │  Bia: "📅 Já existe compromisso  │
│  Google Calendar         │   │        nesse horário. Quer que   │
│                          │   │        eu sugira outro?"         │
└───────────┬──────────────┘   └──────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────┐
│  Bia: "Seu horário foi agendado com sucesso para o dia          │
│        XX/XX/XXXX às XX:XX. Você receberá um lembrete antes     │
│        do horário. Caso precise reagendar ou cancelar, entre    │
│        em contato comigo. Até breve!"                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Tratamento de Erros

### Estratégia de Retry

| Componente | Tentativas | Intervalo | Estratégia |
|------------|------------|-----------|------------|
| **AI Agent** | 2 | 500ms | `continueErrorOutput` |
| **Transcribe a recording** | 2 | 500ms | `continueErrorOutput` |
| **Google Calendar Operations** | 2 | 500ms | `continueErrorOutput` |
| **Insert messages_log** | 3 | 500ms | `continueRegularOutput` |
| **Insert Calendar Events** | 3 | 500ms | `continueErrorOutput` |
| **Insert errors** | 2 | 500ms | `continueRegularOutput` |
| **HTTP Request (Z-API)** | 2 | 500ms | `continueErrorOutput` |

### Fluxo de Erro

```
┌─────────────────────────────────────────────────────────────────┐
│                         ERRO DETECTADO                          │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Error Parser Node                                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Extrai:                                                    │  │
│  │ - source: URL da execução                                  │  │
│  │ - message_id: ID do workflow                               │  │
│  │ - error_code: ID da execução                               │  │
│  │ - error_message: Mensagem de erro                          │  │
│  │ - stack_trace: Stack trace completo                        │  │
│  │ - raw: Payload completo (JSONB)                            │  │
│  │ - created_at: Timestamp                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Insert errors (PostgreSQL)                                     │
│  - Persiste erro na tabela `errors`                            │
│  - `resolved: false` por padrão                                │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Error Friendly (HTTP Request para Z-API)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ "Eita! Me enrolei um pouco aqui e não entendi sua         │  │
│  │  mensagem. 🤔 Consegue repetir pra mim? Se for para       │  │
│  │  agendar, mudar o dia e a hora... Se precisar de algo,    │  │
│  │  me avisa!"                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Mensagens de Erro por Contexto

| Contexto | Mensagem |
|----------|----------|
| **Erro Geral** | "Eita! Me enrolei um pouco aqui e não entendi sua mensagem. 🤔 Consegue repetir pra mim?" |
| **Evento Não Encontrado** | "Não foi possível localizar o evento para alteração. Ele pode ter sido cancelado anteriormente..." |
| **Horário Ocupado** | "📅 Já existe compromisso(s) nesse horário. Quer que eu sugira outro horário?" |
| **Áudio Não Transcrito** | "Ops! Tive um problema para entender seu áudio agora. Pode tentar novamente ou mandar em texto?" |

---

## 🧪 Testes Realizados

### Casos de Teste Obrigatórios

| # | Caso de Teste | Input | Expected Output | Status |
|---|---------------|-------|-----------------|--------|
| 1 | Áudio criar evento | "Bia, marquei uma reunião às 18:00 com o fulano" | Transcrição + Confirmação + Criação | ✅ |
| 2 | Áudio alterar evento | "Bia, muda a reunião das 18:00 para 19:00" | Update com `match_strategy: by_time` | ✅ |
| 3 | Áudio cancelar evento | "Bia, cancela a reunião com o fulano" | Cancel com `match_strategy: by_person` | ✅ |
| 4 | Texto consultar status | "Bia, status do meu agendamento" | Lista de eventos ou "agenda livre" | ✅ |
| 5 | Idempotência | Reenvio do mesmo `message_id` | Ignorar (não duplicar) | ✅ |
| 6 | Falha Gemini | Credencial inválida | Retry + Erro amigável + Registro DB | ✅ |

### Matriz de Validação

| Critério | Peso | Implementação | Status |
|----------|------|---------------|--------|
| **Áudio fim-a-fim** | 25% | HTTP Request → Gemini Transcribe → AI Agent → Calendar | ✅ |
| **IA/JSON** | 20% | Schema v1 + Confirmação + Clarificação | ✅ |
| **Integrações** | 20% | Calendar + Z-API + PostgreSQL | ✅ |
| **Histórico** | 15% | messages_log + llm_runs + calendar_events | ✅ |
| **Retry + Erro** | 10% | 2-3 tentativas + mensagem amigável | ✅ |
| **Idempotência** | 10% | Check Duplicate via message_id | ✅ |

### Evidências

#### Google Calendar com Evento Criado

```
📅 Reunião com Maria Vitória
📍 Quinta-feira, 8 de janeiro • 5:00 – 6:00pm
🔔 30 minutos antes
📂 Compromissos
```

#### Logs no Supabase

**messages_log:**
```json
{
  "message_id": "3EB0B4E5B89BE8F7CBC6C9",
  "phone": "5543991234567",
  "direction": "inbound",
  "message_type": "audio",
  "transcript": "Bia, marca uma reunião com Maria Vitória quinta às 17h",
  "processed": true
}
```

**llm_runs:**
```json
{
  "model": "gemini-2.5-flash",
  "prompt_version": "v1",
  "success": true,
  "output_json": {
    "action": "create",
    "confidence": 0.95,
    "event": {
      "title": "Reunião com Maria Vitória",
      "start_datetime": "2026-01-08T17:00:00-03:00"
    }
  }
}
```

---

## 🐛 Troubleshooting

### Problemas Comuns e Soluções

#### 1. Loop infinito de mensagens

**Sintoma**: Bia responde e processa a própria resposta infinitamente.

**Causa**: Falta de filtro `fromMe === false`.

**Solução**: Verificar o nó Filter com condição:
```javascript
{{ $json.body.fromMe }} equals false
```

#### 2. Webhook não recebe mensagens

**Sintoma**: Mensagens do WhatsApp não chegam ao n8n.

**Causa**: ngrok expirou ou URL mudou.

**Solução**:
1. Reinicie o ngrok: `ngrok http 5555`
2. Atualize a URL no Z-API (Settings → Webhooks)
3. Atualize `WEBHOOK_URL` no docker-compose
4. Reinicie o container: `docker-compose restart`

#### 3. Erro de autenticação Google Calendar

**Sintoma**: `Error: invalid_grant` ou `Token has been revoked`.

**Causa**: Token OAuth expirado ou revogado.

**Solução**:
1. Vá em **n8n** → **Settings** → **Credentials**
2. Edite a credencial Google Calendar
3. Clique em **Reconnect**
4. Autorize novamente no Google

#### 4. Mensagens duplicadas processadas

**Sintoma**: Mesmo evento criado múltiplas vezes.

**Causa**: Check Duplicate não funcionando.

**Solução**:
1. Verifique índice único em `messages_log`:
```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_messages_log_message_id 
ON messages_log(message_id);
```
2. Verifique query no nó Check Duplicate

#### 5. IA retorna UPDATE ao invés de CREATE

**Sintoma**: Ao corrigir sugestão, IA usa `action: update`.

**Causa**: Prompt não distingue sugestão de evento existente.

**Solução**: O prompt atual inclui regras 6, 7 e 8 para distinguir:
- Sugestão (ainda não criado) → `create`
- Evento existente (já criado com ✅) → `update`

#### 6. Áudio não transcreve

**Sintoma**: `Error: Unable to transcribe audio`.

**Causa**: Formato de áudio incompatível ou quota excedida.

**Solução**:
1. Verifique quota da API Gemini no Google Cloud Console
2. Verifique se o áudio está em formato suportado (OGG, MP3, WAV)
3. Verifique logs do nó HTTP Request para URL válida

#### 7. Eventos criados com timezone errado

**Sintoma**: Evento aparece em horário diferente no Google Calendar.

**Causa**: Datetime sem timezone explícito.

**Solução**: Garantir que todas as datas usam `-03:00`:
```javascript
"2026-01-06T14:00:00-03:00"
```

---

## 🔐 Variáveis de Ambiente

### Arquivo `.env.example`

```env
# ═══════════════════════════════════════════════════════════════════
# n8n Configuration
# ═══════════════════════════════════════════════════════════════════
N8N_ENCRYPTION_KEY=your_32_character_encryption_key_here
N8N_EDITOR_BASE_URL=https://your-domain.ngrok-free.dev
WEBHOOK_URL=https://your-domain.ngrok-free.dev
N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true
N8N_DEFAULT_BINARY_DATA_MODE=filesystem

# ═══════════════════════════════════════════════════════════════════
# PostgreSQL / Supabase Configuration
# ═══════════════════════════════════════════════════════════════════
POSTGRES_HOST=aws-1-sa-east-1.pooler.supabase.com
POSTGRES_PORT=5432
POSTGRES_DATABASE=postgres
POSTGRES_USER=postgres.your_project_id
POSTGRES_PASSWORD=your_supabase_password

# ═══════════════════════════════════════════════════════════════════
# Google Cloud Configuration
# ═══════════════════════════════════════════════════════════════════
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_GEMINI_API_KEY=your_gemini_api_key
GOOGLE_CALENDAR_ID=your_email@gmail.com

# ═══════════════════════════════════════════════════════════════════
# Z-API Configuration
# ═══════════════════════════════════════════════════════════════════
ZAPI_INSTANCE_ID=your_instance_id
ZAPI_TOKEN=your_token
ZAPI_CLIENT_TOKEN=your_client_token
ZAPI_BASE_URL=https://api.z-api.io/instances

# ═══════════════════════════════════════════════════════════════════
# ngrok Configuration
# ═══════════════════════════════════════════════════════════════════
NGROK_AUTHTOKEN=your_ngrok_authtoken
NGROK_DOMAIN=your-domain.ngrok-free.dev
```

### Credenciais Necessárias no n8n

| Credencial | Tipo | Campos Necessários | Onde Obter |
|------------|------|-------------------|------------|
| **PostgreSQL** | Postgres | Host, Database, User, Password, Port | Supabase Dashboard → Settings → Database |
| **Google Calendar** | OAuth2 | Client ID, Client Secret | Google Cloud Console → APIs & Services → Credentials |
| **Google Gemini** | API Key | API Key | Google AI Studio → Get API Key |

---

## 🚀 Melhorias Futuras

### Backlog de Funcionalidades

- [ ] **Filtro avançado de eventos** - Implementar `match_strategy` completo para update/cancel
- [ ] **Múltiplos calendários** - Suporte a seleção de calendário pelo usuário
- [ ] **Notificações de lembrete** - Mensagens automáticas antes dos eventos
- [ ] **Google Meet** - Links de videoconferência automáticos
- [ ] **Dashboard de métricas** - Visualização de uso e erros
- [ ] **Recorrência** - Eventos semanais/mensais
- [ ] **Cancelamento em lote** - "Cancela todos os eventos de hoje"
- [ ] **Multi-idioma** - Suporte a EN, ES além de PT-BR
- [ ] **Rate limiting** - Proteção contra flood de mensagens
- [ ] **Fila de processamento** - Redis/BullMQ para alta disponibilidade

### Otimizações Técnicas

- [ ] **Cache de disponibilidade** - Redis para FreeBusy frequentes
- [ ] **Webhook assíncrono** - Resposta imediata + processamento em background
- [ ] **Logs estruturados** - JSON logs para Elasticsearch/Loki
- [ ] **Health checks** - Endpoint para monitoramento
- [ ] **Backup automático** - Exportação periódica do workflow

---

## 📄 Credenciais e Segurança

### Boas Práticas Implementadas

| Prática | Implementação |
|---------|---------------|
| **Sem hardcode de tokens** | Uso de credenciais n8n |
| **Variáveis de ambiente** | docker-compose com env vars |
| **Conexão segura** | HTTPS via ngrok |
| **OAuth2** | Google Calendar com refresh token |
| **Idempotência** | Prevenção de duplicatas |
| **Logs de auditoria** | Registro completo de operações |

### Dados Sensíveis

> ⚠️ **ATENÇÃO**: Nunca commitar no repositório:
> - Arquivos `.env` com valores reais
> - Exports de workflow com credenciais
> - API Keys ou tokens
> - Connection strings

---

## 👤 Autor

Desenvolvido como parte de um teste técnico para demonstrar habilidades em:

- ✅ Automação com n8n (79 nós)
- ✅ Integração de APIs (Google Calendar, Z-API, Gemini)
- ✅ IA Conversacional (Prompt Engineering)
- ✅ Arquitetura de sistemas (Microservices pattern)
- ✅ Banco de dados (PostgreSQL/Supabase)
- ✅ DevOps (Docker, ngrok)

---

## 📜 Licença

Este projeto é apenas para fins de demonstração e avaliação técnica.

---

<p align="center">
  <strong>🤖 Bia - Sua secretária virtual inteligente</strong><br>
  <em>"Agendar, alterar, cancelar - é só chamar!"</em>
</p>
