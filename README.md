# 🟣 Agentic Productivity System with MCP

> Cognitive executive assistant with persistent memory, multimodal processing, and sub-agent orchestration via MCP Protocol.

[![n8n](https://img.shields.io/badge/n8n-Workflow-FF6D5A?logo=n8n)](https://n8n.io)
[![Telegram](https://img.shields.io/badge/Telegram-Bot-26A5E4?logo=telegram)](https://telegram.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Vector-336791?logo=postgresql)](https://postgresql.org)
[![License](https://img.shields.io/badge/License-Proprietary-red)]()

---

## 📋 Overview

![Orchestrator](assets/orchestrator.png)

**Mira** is an AI-based orchestrator that centralizes Google Workspace services (Calendar, Tasks, Gmail) and financial management into a conversational interface on Telegram. The system implements a cognitive architecture inspired by the human memory model, featuring sensory processing, short-term memory, and consolidation into long-term memory.

### Key Capabilities

* 🧠 **Cognitive Architecture**: Clear separation between sensory, short-term, and long-term memory.
* 🎙️ **Multimodal**: Processes text, audio, images, and documents via Google Gemini 2.0.
* 🔒 **Guardrails**: Detection of NSFW content and jailbreak attempts.
* 🔧 **MCP Protocol**: Specialized sub-agents for specific tasks.
* 📊 **RAG System**: Retrieval-Augmented Generation with Supabase Vector Store.
* ⚡ **Smart Buffer**: Message aggregation for conversational context preservation.

---

## 🏗️ System Architecture

### High-Level Overview

```mermaid
graph TB
    subgraph "Input Layer"
        TG[Telegram Bot]
        USER[User]
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

## 🧩 Technical Components

### 1. Sensory Layer (Input Processing)

![Sensory Identifier](assets/sensory-identifier.png)

**Responsibility**: Identification and normalization of multimodal inputs.

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

* **Google Gemini 2.0 Flash**: Audio transcription, image analysis, and document extraction.
* **Llama 3.1 70B**: Guardrails (NSFW detection, jailbreak prevention).
* **Threshold**: 0.7 for both guardrails.

**Metrics:**

* Average Latency: 800ms - 1.5s
* Accuracy (guardrails): ~94%

---

### 2. Sensory Memory (Message Buffer)

![Buffer](assets/sensory-memory.png)

**Responsibility**: Aggregation of sequential messages to build context.

**Algorithm:**

```sql
-- 1. Insert into buffer
INSERT INTO message_buffer (chat_id, content, batch_id)
VALUES ($chat_id, $content, NULL);

-- 2. Wait 3 seconds (allows for multiple incoming messages)

-- 3. Atomic marking with batch_id
UPDATE message_buffer
SET batch_id = $execution_id
WHERE chat_id = $chat_id 
  AND batch_id IS NULL
RETURNING content;

-- 4. Aggregation
SELECT STRING_AGG(content, '\n' ORDER BY id) as full_context
FROM message_buffer
WHERE batch_id = $execution_id;

-- 5. Post-processing Cleanup
DELETE FROM message_buffer WHERE batch_id = $execution_id;

```

**Advantages:**

* ✅ **Atomicity**: Usage of `batch_id` prevents race conditions.
* ✅ **Context Window**: Multiple messages within ~3s are processed together.
* ✅ **Automatic Cleanup**: Buffer is cleared after each cycle.

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

* **Context Window**: 10 messages (Short-term Memory).
* **Temperature**: Default (0.7).
* **Built-in**: Web Search (medium context).

#### Prompt Engineering

**Applied Strategies:**

1. **Chain-of-Thought (CoT)**: Mandatory `think` tool for explicit reasoning.
2. **Few-Shot Learning**: Interaction examples embedded in the system prompt.
3. **TOON (Token Oriented Object Notation)**: Hierarchical prompt structuring.
4. **Tool Calling**: Decision-making based on intent analysis.

**System Prompt Structure:**

```
🟣 SYSTEM_IDENTITY
🟣 CONTEXT_VARIABLES (date, time, user)
🟣 GLOBAL_CONSTRAINTS (formatting, data integrity)
🟣 DECISION_PROTOCOL (priority order)
🟣 TOOL_REGISTRY (tech specs)
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

**Retention Policy:**

* **Active Window**: Last 10 messages.
* **Cleanup**: Messages > 30 days are deleted (monthly cron).

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
        SCHEMA --> T[main_topic]
        SCHEMA --> E[entities]
        SCHEMA --> A[action_taken]
        SCHEMA --> I[relevant_info]
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
-- 24h Aggregation
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

* **Embedding Model**: `text-embedding-3-small` (OpenAI).
* **Distance Metric**: Cosine Similarity.
* **Top-K**: 5 results.
* **Metadata Filtering**: `chat_id`, `date_range`.

---

### 5. MCP Sub-agents (Task Delegation)

![sub-agents](assets/sub-agent-mcp.png)

**MCP Protocol**: Model Context Protocol used for communication between the main agent and specialized sub-agents.

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
| --- | --- | --- | --- |
| `calendar_agent` | CRUD events, list, search | Google Calendar | - |
| `gmail_agent` | Send, reply, label, search | Gmail | - |
| `financial_agent` | Log expenses, read balance | Google Sheets | `personal or business` |
| `financial_report` | Generate charts, summaries | Google Sheets + Chart.js | `personal or business` |
| `tasks_agent` | CRUD tasks, mark complete | Google Tasks | - |

**MCP Call Example:**

```json
{
  "tool": "sub_agents",
  "params": {
    "agent": "calendar_agent",
    "prompt": "Schedule meeting with Ana on Jan 15th, 2026 at 2 PM",
    "scope": null
  }
}

```

**Response Handling:**

* **Success**: Sub-agent returns a structured confirmation.
* **Failure**: Automatic retry (max 2 attempts).
* **Media Output**: `financial_report` returns an image (bypassing text generation).

---

### Error Handling

![Error Handling](assets/error-handling.png)

This system implements a robust error handling mechanism to ensure continuous execution and explicit recovery. Specifically, it uses an **Error Trigger** in n8n to detect failures and unblock the current flow state.

#### Implemented Error Flows

##### 1. Flow Unblocking Mechanism

An **Error Trigger** is activated if a problem occurs in the execution associated with the `message_buffer`. The flow voids the current batch to prevent deadlocks and reprocesses messages:

Flow:

1. **Trigger**: Detects error event.
2. **Unclogger**: Removes `batch_id` from `message_buffer` using the following SQL:
```sql
UPDATE message_buffer
SET batch_id = NULL
WHERE batch_id = '{{ $execution.id }}';

```



This process ensures no message remains locked, allowing new executions for the affected flow.

##### 2. Short-Term Memory Cleanup

A regularly scheduled job (**Scheduled Trigger**) deletes obsolete records (interactions older than 30 days):

Flow:

1. **Trigger**: Runs monthly at 3:00 AM.
2. **Cleaner**: Executes the following command:
```sql
DELETE FROM n8n_chat_histories 
WHERE created_at < NOW() - INTERVAL '30 days';

```



This ensures optimal performance by preserving only the relevant window for `STM` operations.

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

| Scenario | Latency | Tokens | Cost (estimated) |
| --- | --- | --- | --- |
| Simple text (no tools) | ~3s | 1k-3k | $0.001-0.003 |
| Text + tool calling | ~7s-10s | 4k-15k | $0.004-0.015 |

### Memory Statistics

* **Short-term Window**: 10 messages (rolling).
* **Long-term Storage**: ~30 memories/month.

---

## 🔧 Tech Stack

### Core Infrastructure

* **Orchestration**: n8n (self-hosted)
* **Database**: PostgreSQL 15 + pgvector
* **Vector Store**: Supabase (managed)
* **Hosting**: Hostinger

### AI Models

| Component | Model | Provider | Purpose |
| --- | --- | --- | --- |
| Main Agent | GPT-4.1-mini | OpenAI | Cognitive orchestration |
| Transcription | Gemini 2.0 Flash | Google | Audio → Text |
| Image Analysis | Gemini 2.0 Flash | Google | Vision → Text |
| Document Analysis | Gemini 2.0 Flash | Google | PDF/Doc → Text |
| Guardrails | Llama 3.1 70B | OpenRouter | Safety checks |
| Memory Extraction | Llama 3.3 70B | OpenRouter | Information extraction |
| Embeddings | text-embedding-3-small | OpenAI | Vector generation |

### Integrations

* **Telegram Bot API**: User interface
* **Google Cloud Platform**:
* Calendar API
* Gmail API
* Tasks API
* Sheets API


* **MCP Protocol**: Custom sub-agent server

---

## 🚀 Use Cases

### 1. Agenda Management

```
👤 User: "Lunch with Ana tomorrow at 1 PM"
🟣 Mira: [Calls calendar_agent]
       "Done! Scheduled Lunch with Ana 
        for tomorrow (Jan 15) at 1 PM. ✅"

```

### 2. Financial Control

```
👤 User: "Spent $50 on lunch"
🟣 Mira: "Was this Personal or Business expense?"
👤 User: "Business"
🟣 Mira: [Calls financial_agent]
       "Logged! 💰 $50.00 (Business - Food)"

```

### 3. Visual Reporting

```
👤 User: "How much did I spend this month?"
🟣 Mira: "Do you want the Personal or Business report?"
👤 User: "Personal"
🟣 Mira: [Calls financial_report]
       [Sends PNG chart via Telegram]

```

### 4. Context Retrieval

```
👤 User: "What did I agree with Carlos in the last meeting?"
🟣 Mira: [Searches long-term memory]
       "In the meeting on Jan 10th, you agreed with Carlos to:
        • Deliver the proposal by Jan 20th
        • Review the cost spreadsheet
        • Next meeting: Jan 25th at 3 PM"

```

---

## 📄 Full Technical Documentation

This README presents the high-level architecture of the project. For access to the complete technical documentation, including:

* 🔧 Setup guide with mock credentials
* 📊 Detailed cost analysis
* 🎥 Video demos of use cases
* 📝 Sanitized Workflow JSON
* 🧪 Performance tests

**Contact me** via codeajr@gmail.com.

---

## 🤝 Contributions

This is a proprietary project developed for personal/commercial use. The full source code is not publicly available, but technical suggestions and discussions are welcome via Issues.

---

## 📝 License

**Proprietary License** - All rights reserved.

This project is confidential and contains proprietary integrations. This documentation is shared solely for technical portfolio purposes.

---

## 👤 Author

**André Codea**

* LinkedIn: [https://linkedin.com/in/andrecodea](https://linkedin.com/in/andrecodea)
* GitHub: [https://github.com/andrecodea](https://github.com/andrecodea)
* Email: codeajr@gmail.com

---

<p align="center">
<i>Built with ❤️ using n8n, OpenAI, and lots of ☕</i>
</p>
