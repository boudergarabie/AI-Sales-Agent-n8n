# Architecture

The system is split into eight small n8n workflows. This page shows how they fit together.

## 1. Big picture

```mermaid
flowchart LR
    C(["Customer<br/>on Messenger"]) --> I["01 Intake & routing"]
    I --> P{"Bot paused for<br/>this customer?"}
    P -- yes --> H["A human handles the chat"]
    P -- no --> M["Understand the message"]
    M --> B["Batch rapid messages<br/>(Redis)"]
    B --> A["02 Sales agent"]
    A --> R["Reply: text, photos, voice"]
    R --> C
    A -. "needs a human" .-> E["04 Escalation"]
    E --> T["Telegram alert"]
    ERR["07 Error handler"] -. "any workflow fails" .-> T
```

## 2. Understanding what the customer sent

Workflow 01 detects the message type and turns everything into text before the agent sees it.

```mermaid
flowchart TB
    W["Messenger webhook"] --> X{"Type of message"}
    X -- "text" --> TXT["Use the text as is"]
    X -- "voice note" --> STT["Groq Whisper<br/>(prompted for Darija)"]
    X -- "image" --> IMG["Gemini vision"]
    X -- "video" --> VID["Gemini vision"]
    X -- "shared post" --> POST["Graph API: post details<br/>+ image analysis"]
    X -- "shared reel" --> REEL["Graph API: reel details<br/>+ video analysis"]
    TXT --> OUT["One combined message<br/>for the agent"]
    STT --> OUT
    IMG --> OUT
    VID --> OUT
    POST --> OUT
    REEL --> OUT
```

Messages the store owner sends manually from the Page inbox arrive as "echo" events. They are detected and stored in the conversation memory instead of being answered.

**Batching.** Each customer has a short-lived buffer in Redis. Messages are appended to it, a lock makes sure only one run answers, the run waits 20 seconds, then reads the whole buffer and answers once.

## 3. The sales agent and its tools

```mermaid
flowchart LR
    A["Sales agent<br/>(LLM + conversation memory)"]
    A --> T1["Product<br/>catalog sheet"]
    A --> T2["Inventory<br/>stock by size and color"]
    A --> T3["Yalidine_Tarifs<br/>delivery prices"]
    A --> T4["Knowledge base<br/>store policies (vector search)"]
    A --> T5["Orders<br/>register a confirmed order"]
    A --> T6["Update_Inventory_Reserved<br/>reserve the stock"]
    A --> T7["escalate_to_human"]
```

Order closing follows a fixed sequence enforced by the prompt: confirm the exact variant and quantity, collect customer details, show the total (product price plus real delivery fee), get explicit confirmation, register the order, then reserve the stock.

After the agent answers, a formatting step extracts product image links, and the reply is sent as text, as product photos, or as a voice note (generated with ElevenLabs) when the customer sent a voice note.

## 4. Human handoff

```mermaid
sequenceDiagram
    participant C as Customer
    participant A as Sales agent
    participant E as Escalation (04)
    participant S as Bot_Control sheet
    participant O as Store owner (Telegram)
    C->>A: Asks to talk to a person
    A->>E: escalate_to_human
    E->>E: Summarize the last messages (Algerian Arabic)
    E->>S: bot_paused = true for this customer
    E->>O: Alert with summary and ON / OFF buttons
    O->>S: Taps a button (workflow 05)
    Note over S: A scheduled check (workflow 06) re-enables the bot after 12 hours
```

## 5. Error monitoring

```mermaid
flowchart LR
    F["Any workflow fails"] --> ET["07 Error trigger"]
    ET --> P["Parse the error"]
    P --> D["LLM diagnosis:<br/>cause + suggested fix"]
    D --> TG["Telegram alert"]
    D --> DB["Row in error_logs"]
```

## 6. Where data lives

| Data | Where | Why |
|------|-------|-----|
| Catalog, stock, delivery tariffs, orders | Google Sheets | Easy for a shop owner to edit |
| Bot on/off per customer | Google Sheets (`Bot_Control`) | Visible and editable by the owner |
| Conversation memory | Postgres | Persistent context per customer |
| Message buffer and lock | Redis | Fast, expires automatically |
| Store policies | Supabase vector store | Semantic search instead of guessing |
| Error log | Postgres (`error_logs`) | History of failures with the AI diagnosis |
