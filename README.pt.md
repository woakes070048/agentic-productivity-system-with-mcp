# 🟣 Agentic Productivity System with MCP


[Demo](https://www.linkedin.com/posts/andrecodea_artificialintelligence-n8n-aiautomation-activity-7402859317948768256-7zIF)

> Assistente executivo cognitivo com memória persistente, processamento multimodal e orquestração de sub-agentes via MCP Protocol.

[![n8n](https://img.shields.io/badge/n8n-Workflow-FF6D5A?logo=n8n)](https://n8n.io)
[![Telegram](https://img.shields.io/badge/Telegram-Bot-26A5E4?logo=telegram)](https://telegram.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Vector-336791?logo=postgresql)](https://postgresql.org)
[![License](https://img.shields.io/badge/License-Proprietary-red)]()

---

## 📋 Visão Geral

**Mira** é a orquestradora baseada em IA que centraliza serviços do Google Workspace (Calendar, Tasks, Gmail) e gerenciamento financeiro em uma interface conversacional no Telegram. O sistema implementa uma arquitetura cognitiva inspirada no modelo de memória humana, com processamento sensorial, memória de curto prazo e consolidação para memória de longo prazo.

![Orquestrador](assets/orchestrator.png)

### Características Principais

- 🧠 **Arquitetura Cognitiva**: Separação clara entre sensory memory, short-term e long-term memory
- 🎙️ **Multimodal**: Processa texto, áudio, imagens e documentos via Google Gemini 2.0
- 🔒 **Guardrails**: Detecção de conteúdo NSFW e tentativas de jailbreak
- 🔧 **MCP Protocol**: Sub-agentes especializados para tarefas específicas
- 📊 **RAG System**: Retrieval-Augmented Generation com Supabase Vector Store
- ⚡ **Buffer Inteligente**: Agregação de mensagens para contexto conversacional

---

## 🏗️ Arquitetura do Sistema

### High-Level Overview

```mermaid
graph TB
    subgraph "Input Layer"
        TG[Telegram Bot]
        USER[Usuário]
    end
    
    subgraph "Sensory Processing"
        SWITCH{Content Type}
        AUDIO[Audio Transcription]
        IMAGE[Image Analysis]
        DOC[Document Analysis]
        TEXT[Text Input]
        GR[Guardrails<br/>NSFW + Jailbreak]
    end
    
    subgraph "Sensory Memory"
        BUFFER[(Message Buffer<br/>PostgreSQL)]
        WAIT[Wait 3s]
        AGG[Message Aggregator]
    end
    
    subgraph "Cognitive Layer"
        AGENT[Complex Agent<br/>GPT-4.1-mini]
        STM[(Short-term Memory<br/>PostgreSQL)]
        LTM[(Long-term Memory<br/>Vector Store)]
    end
    
    subgraph "Tool Registry"
        THINK[Think Tool]
        CALC[Calculator]
        MCP[MCP Sub-agents]
        SEARCH[Web Search]
    end
    
    subgraph "Output Layer"
        SEND[Telegram Send]
        CLEAN[Buffer Cleanup]
    end
    
    USER -->|Message| TG
    TG --> SWITCH
    SWITCH -->|Text| TEXT
    SWITCH -->|Audio| AUDIO
    SWITCH -->|Image| IMAGE
    SWITCH -->|Document| DOC
    
    TEXT --> GR
    AUDIO --> GR
    IMAGE --> GR
    DOC --> GR
    
    GR -->|Safe| BUFFER
    GR -->|Unsafe| SEND
    
    BUFFER --> WAIT
    WAIT --> AGG
    AGG --> AGENT
    
    AGENT <--> STM
    AGENT <--> LTM
    AGENT <--> THINK
    AGENT <--> CALC
    AGENT <--> MCP
    AGENT <--> SEARCH
    
    AGENT --> SEND
    SEND --> CLEAN
    CLEAN --> BUFFER
```

---

## 🧩 Componentes Técnicos

### 1. Sensory Layer (Input Processing)

![Sensory Identifier](assets/sensory-identifier.png)

**Responsabilidade**: Identificação e normalização de inputs multimodais.

```mermaid
graph LR
    INPUT[Input] --> SWITCH{Type?}
    SWITCH -->|text| TEXT[Direct to Guardrails]
    SWITCH -->|voice| VOICE[Get Audio File]
    SWITCH -->|photo| PHOTO[Get Image File]
    SWITCH -->|document| DOC[Get Document File]
    
    VOICE --> TRANS[Transcribe<br/>Gemini 2.0]
    PHOTO --> ANALYZE[Analyze Image<br/>Gemini 2.0]
    DOC --> EXTRACT[Extract Text<br/>Gemini 2.0]
    
    TRANS --> GR[Guardrails]
    ANALYZE --> GR
    EXTRACT --> GR
    TEXT --> GR
    
    GR -->|Pass| BUFFER[(Buffer)]
    GR -->|Fail| REJECT[Send Rejection]
```

**Stack:**
- **Google Gemini 2.0 Flash**: Transcrição de áudio, análise de imagens e extração de documentos
- **Llama 3.1 70B**: Guardrails (NSFW detection, jailbreak prevention)
- **Threshold**: 0.7 para ambos os guardrails

**Métricas:**
- Latência média: 800ms - 1.5s
- Accuracy (guardrails): ~94%

---

### 2. Sensory Memory (Message Buffer)

![Buffer](assets/sensory-memory.png)

**Responsabilidade**: Agregação de mensagens sequenciais para construção de contexto.

**Algoritmo:**
```sql
-- 1. Inserção no buffer
INSERT INTO message_buffer (chat_id, content, batch_id)
VALUES ($chat_id, $content, NULL);

-- 2. Wait 3 segundos (permite múltiplas mensagens)

-- 3. Marcação atômica com batch_id
UPDATE message_buffer
SET batch_id = $execution_id
WHERE chat_id = $chat_id 
  AND batch_id IS NULL
RETURNING content;

-- 4. Agregação
SELECT STRING_AGG(content, '\n' ORDER BY id) as full_context
FROM message_buffer
WHERE batch_id = $execution_id;

-- 5. Limpeza pós-processamento
DELETE FROM message_buffer WHERE batch_id = $execution_id;
```

**Vantagens:**
- ✅ **Atomicidade**: Uso de `batch_id` evita race conditions
- ✅ **Context Window**: Múltiplas mensagens em ~3s são processadas juntas
- ✅ **Cleanup Automático**: Buffer limpo após cada ciclo

---

### 3. Cognitive Layer (Agent + Memory)

![Agent](assets/agent.png)

#### Agent Architecture

```mermaid
graph TB
    subgraph "Agent Core"
        INPUT[User Input] --> THINK[Think Tool<br/>Intent Analysis]
        THINK --> DECISION{Decision Type}
    end
    
    subgraph "Memory Systems"
        STM[(Short-term<br/>PostgreSQL<br/>10 msgs)]
        LTM[(Long-term<br/>Supabase Vector<br/>OpenAI Embeddings)]
    end
    
    subgraph "Tool Registry"
        CALC[Calculator]
        WEB[Web Search<br/>Native GPT-4.1]
        MCP[MCP Sub-agents]
    end
    
    DECISION -->|Retrieval| LTM
    DECISION -->|Action| MCP
    DECISION -->|Compute| CALC
    DECISION -->|Research| WEB
    
    STM -.->|Context| DECISION
    LTM -.->|Memories| DECISION
    
    MCP --> OUTPUT[Response]
    CALC --> OUTPUT
    WEB --> OUTPUT
    LTM --> OUTPUT
```

**Model:** GPT-4.1-mini (gpt-5.1)
- **Context Window**: 10 mensagens (Short-term Memory)
- **Temperature**: Default (0.7)
- **Built-in**: Web Search (medium context)

#### Prompt Engineering

**Estratégias aplicadas:**
1. **Chain-of-Thought (CoT)**: Tool `think` obrigatória para raciocínio explícito
2. **Few-Shot Learning**: Exemplos de interações no system prompt
3. **TOON (Token Oriented Object Notation)**: Estruturação hierárquica do prompt
4. **Tool Calling**: Decisão baseada em intent analysis

**System Prompt Structure:**
```
🟣 SYSTEM_IDENTITY
🟣 CONTEXT_VARIABLES (date, time, user)
🟣 GLOBAL_CONSTRAINTS (formatting, data integrity)
🟣 DECISION_PROTOCOL (priority order)
🟣 TOOL_REGISTRY (specs técnicas)
🟣 ORCHESTRATION_PROTOCOL (workflow)
🟣 FEW_SHOT_EXAMPLES
```

---

### 4. Memory Systems

#### Short-term Memory (Working Memory)

![stm](assets/stm.png)

```mermaid
graph LR
    A[New Interaction] --> B[(PostgreSQL<br/>n8n_chat_histories)]
    B --> C{Window Size}
    C -->|Keep| D[Last 10 messages]
    C -->|Archive| E[Long-term Consolidation]
    D --> F[Agent Context]
```

**Schema:**
```sql
CREATE TABLE n8n_chat_histories (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(255),
    message JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Política de Retenção:**
- **Active Window**: 10 últimas mensagens
- **Cleanup**: Mensagens > 30 dias deletadas (monthly cron)

#### Long-term Memory (Episodic Memory)

![ltm](assets/ltm.png)

```mermaid
graph TB
    subgraph "Daily Consolidation (3AM)"
        CRON[Schedule Trigger] --> AGG[Aggregate 24h Messages]
        AGG --> EXTRACT[Information Extractor<br/>Llama 3.3 70B]
    end
    
    subgraph "Extraction Schema"
        EXTRACT --> SCHEMA{Extracted Fields}
        SCHEMA --> T[tema_principal]
        SCHEMA --> E[entidades]
        SCHEMA --> A[acao_tomada]
        SCHEMA --> I[informacao_relevante]
    end
    
    subgraph "Vector Storage"
        SCHEMA --> EMBED[OpenAI Embeddings<br/>text-embedding-3-small<br/>1536 dims]
        EMBED --> VDB[(Supabase pgvector<br/>agent_memory)]
    end
    
    subgraph "Retrieval"
        QUERY[User Query] --> QEMBED[Embed Query]
        QEMBED --> SEARCH[Cosine Similarity]
        VDB --> SEARCH
        SEARCH --> CONTEXT[Top-K Results]
    end
```

**Consolidation Query:**
```sql
-- Agregação de 24h
SELECT STRING_AGG(message->>'content', E'\n' ORDER BY id) as batch
FROM n8n_chat_histories
WHERE created_at > NOW() - INTERVAL '1 day';
```

**Vector Store Schema:**
```sql
CREATE TABLE agent_memory (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding VECTOR(1536)
);

CREATE INDEX ON agent_memory 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

**Retrieval Strategy:**
- **Embedding Model**: `text-embedding-3-small` (OpenAI)
- **Distance Metric**: Cosine Similarity
- **Top-K**: 5 results
- **Metadata Filtering**: `chat_id`, `date_range`

---

### 5. MCP Sub-agents (Task Delegation)

![sub-agents](assets/sub-agent-mcp.png)

**MCP Protocol**: Model Context Protocol para comunicação entre agente principal e sub-agentes especializados.

```mermaid
graph TB
    AGENT[Complex Agent] -->|MCP Request| SERVER[MCP Server]
    
    SERVER --> CAL[calendar_agent]
    SERVER --> MAIL[gmail_agent]
    SERVER --> FIN[financial_agent]
    SERVER --> REPORT[financial_report]
    SERVER --> TASK[tasks_agent]
    
    CAL -->|CRUD| GCAL[Google Calendar API]
    MAIL -->|Send/Reply| GMAIL[Gmail API]
    FIN -->|Read/Write| SHEETS[Google Sheets API]
    REPORT -->|Generate Chart| VIZ[Data Visualization]
    TASK -->|CRUD| GTASKS[Google Tasks API]
    
    GCAL --> RESPONSE[MCP Response]
    GMAIL --> RESPONSE
    SHEETS --> RESPONSE
    VIZ --> RESPONSE
    GTASKS --> RESPONSE
    
    RESPONSE --> AGENT
```

**Sub-agents Specs:**

![financial-agent](assets/financial-agent.png)

| Agent | Capabilities | API | Scope |
|-------|-------------|-----|-------|
| `calendar_agent` | CRUD events, list, search | Google Calendar | - |
| `gmail_agent` | Send, reply, label, search | Gmail | - |
| `financial_agent` | Log expenses, read balance | Google Sheets | `personal` \| `business` |
| `financial_report` | Generate charts, summaries | Google Sheets + Chart.js | `personal` \| `business` |
| `tasks_agent` | CRUD tasks, mark complete | Google Tasks | - |

**MCP Call Example:**
```json
{
  "tool": "sub_agents",
  "params": {
    "agent": "calendar_agent",
    "prompt": "Agendar reunião com Ana dia 15/01/2026 às 14h",
    "scope": null
  }
}
```

**Response Handling:**
- **Success**: Sub-agent retorna confirmação estruturada
- **Failure**: Retry automático (max 2 tentativas)
- **Media Output**: `financial_report` retorna imagem (bypassa texto)

---

### Error Handling

![Error Handling](assets/error-handling.png)

Este sistema implementa um mecanismo robusto de tratamento de erros para garantir execução contínua e recuperação explícita. Particularmente, utiliza um **Error Trigger** no n8n para detectar falhas e desbloquear o estado atual do fluxo.

#### Fluxos de Erro Implementados

##### 1. Destravamento do Fluxo

Um **Error Trigger** é ativado caso ocorra um problema na execução associada ao `message_buffer`. O fluxo anula o lote atual para evitar impasses e reprocessa mensagens:

Fluxo:
1. **Trigger**: Detecta evento de erro.
2. **Unclogger**: Remove `batch_id` do `message_buffer` com o seguinte SQL:
   ```sql
   UPDATE message_buffer
   SET batch_id = NULL
   WHERE batch_id = '{{ $execution.id }}';
   ```

Este processo assegura que nenhuma mensagem permanecerá bloqueada, permitindo novas execuções para o fluxo em questão.

##### 2. Limpeza de Memória de Curto Prazo

Regularmente, um job programado (**Scheduled Trigger**) deleta registros obsoletos (interações acima de 30 dias):

Fluxo:
1. **Trigger**: Roda todo mês às 3h da manhã.
2. **Cleaner**: Executa o seguinte comando:
   ```sql
   DELETE FROM n8n_chat_histories 
   WHERE created_at < NOW() - INTERVAL '30 days';
   ```

Dessa forma, o desempenho mantém-se ideal, preservando apenas os 10 últimos registros para operações na `STM`.

---

## 📊 Performance & Metrics

### Latency Breakdown

```mermaid
gantt
    title Execution Timeline (without tools)
    dateFormat SSS
    section Input
    Sensory Processing    :000, 800ms
    Guardrails Check      :800, 400ms
    section Buffer
    Wait Period           :1200, 3000ms
    Message Aggregation   :4200, 200ms
    section Cognitive
    Agent Processing      :4400, 2000ms
    section Output
    Telegram Send         :6400, 300ms
```

| Cenário | Latência | Tokens | Custo (estimado) |
|---------|----------|--------|------------------|
| Texto simples (sem tools) | ~3s | 1k-3k | $0.001-0.003 |
| Texto + tool calling  | ~7s-10s | 4k-15k | $0.004-0.015 |

### Memory Statistics
- **Short-term Window**: 10 mensagens (rolling)
- **Long-term Storage**: ~30 memories/mês

---

## 🔧 Stack Técnica

### Core Infrastructure
- **Orchestration**: n8n (self-hosted)
- **Database**: PostgreSQL 15 + pgvector
- **Vector Store**: Supabase (managed)
- **Hosting**: Hostinger

### AI Models
| Component | Model | Provider | Purpose |
|-----------|-------|----------|---------|
| Main Agent | GPT-4.1-mini | OpenAI | Cognitive orchestration |
| Transcription | Gemini 2.0 Flash | Google | Audio → Text |
| Image Analysis | Gemini 2.0 Flash | Google | Vision → Text |
| Document Analysis | Gemini 2.0 Flash | Google | PDF/Doc → Text |
| Guardrails | Llama 3.1 70B | OpenRouter | Safety checks |
| Memory Extraction | Llama 3.3 70B | OpenRouter | Information extraction |
| Embeddings | text-embedding-3-small | OpenAI | Vector generation |

### Integrations
- **Telegram Bot API**: User interface
- **Google Cloud Platform**:
  - Calendar API
  - Gmail API
  - Tasks API
  - Sheets API
- **MCP Protocol**: Custom sub-agent server

---

## 🚀 Casos de Uso

### 1. Gestão de Agenda
```
👤 User: "Almoço com a Ana amanhã 13h"
🟣 Mira: [Calls calendar_agent]
       "Combinado! Agendei seu Almoço com a Ana 
        para amanhã (15/01) às 13h. ✅"
```

### 2. Controle Financeiro
```
👤 User: "Gastei 50 reais no almoço"
🟣 Mira: "Esse gasto foi Pessoal ou da Empresa?"
👤 User: "Foi da empresa"
🟣 Mira: [Calls financial_agent]
       "Registrado! 💰 R$ 50,00 (Empresa - Alimentação)"
```

### 3. Relatórios Visuais
```
👤 User: "Quanto gastei esse mês?"
🟣 Mira: "Você quer o relatório Pessoal ou de Negócios?"
👤 User: "Pessoal"
🟣 Mira: [Calls financial_report]
       [Envia gráfico PNG via Telegram]
```

### 4. Recuperação de Contexto
```
👤 User: "O que eu combinei com o Carlos na reunião passada?"
🟣 Mira: [Searches long-term memory]
       "Na reunião de 10/01 você combinou com o Carlos:
        • Entregar proposta até 20/01
        • Revisar planilha de custos
        • Próxima reunião: 25/01 às 15h"
```

---

## 📄 Documentação Técnica Completa

Este README apresenta a arquitetura high-level do projeto. Para acesso à documentação técnica completa, incluindo:

- 🔧 Setup guide com credenciais mock
- 📊 Análise de custos detalhada
- 🎥 Video demos de casos de uso
- 📝 Workflow JSON sanitizado
- 🧪 Testes de performance

**Entre em contato** via codeajr@gmail.com.

---

## 🤝 Contribuições

Este é um projeto proprietário desenvolvido para uso pessoal/comercial. O código-fonte completo não está disponível publicamente, mas sugestões e discussões técnicas são bem-vindas via Issues.

---

## 📝 Licença

**Proprietary License** - Todos os direitos reservados.

Este projeto é confidencial e contém integrações proprietárias. A documentação é compartilhada apenas para fins de portfolio técnico.

---

## 👤 Autor

**André Codea**
- LinkedIn: https://linkedin.com/in/andrecodea
- GitHub: https://github.com/andrecodea
- Email: codeajr@gmail.com

---

<p align="center">
  <i>Built with ❤️ using n8n, OpenAI, and lots of ☕</i>
</p>
