# n8nClaw

A lightweight, self-hosted AI assistant built entirely in [n8n](https://n8n.io). Inspired by [OpenClaw](https://github.com/nicepkg/openclaw) — multi-channel messaging, persistent memory, task management, and autonomous work — all in a single visual workflow.

## Architecture

```
                        TRIGGERS
          +-----------+-----------+-----------+
          | Telegram  | WhatsApp  |   Gmail   | Hourly
          | Trigger   | Webhook   |  Trigger  | Heartbeat
          +-----------+-----------+-----------+
                |           |           |           |
          [ Filter ]  [ Filter ]  [ Get User ] [ Get User ]
                |           |           |           |
          [ Get User Profile from Init Table ]
                |           |           |           |
          [ Edit Fields — normalize: user_message, system_prompt, last_channel ]
                \           |           |           /
                 +----------+-----------+----------+
                            |
                      +-----v------+
                      |  n8nClaw   |  (Claude Sonnet 4.5 via OpenRouter)
                      |  AI Agent  |  (Postgres Chat Memory - 15 msgs)
                      +-----+------+
                            |
               +------------+-------------+
               |            |             |
          [ Switch: last_channel ]
               |            |
          Telegram     WhatsApp
          Reply        Reply (Evolution API)

                      TOOLS & SUB-AGENTS
          +-------------------------------------------+
          | Tasks DB    | Subtasks DB  | Init/User DB  |
          | Research    | Email Mgr    | Doc Manager   |
          | Worker 1    | Worker 2     | Worker 3      |
          | Vector Store (Supabase RAG)               |
          +-------------------------------------------+

                    MEMORY PIPELINE (scheduled)
          +-------------------------------------------+
          | Postgres Chat History                      |
          |   -> Aggregate messages                    |
          |   -> Summarize (Haiku 4.5)                 |
          |   -> Embed (OpenAI)                        |
          |   -> Store in Supabase Vector DB           |
          +-------------------------------------------+
```

## What's Included

### Channels (Triggers)
| Channel | Trigger Type | Notes |
|---------|-------------|-------|
| **Telegram** | Native Telegram Trigger | Supports text, voice, images, documents |
| **WhatsApp** | Webhook (Evolution API) | Text messages via Evolution API |
| **Gmail** | Poll trigger (every minute) | Auto-processes incoming emails |
| **Heartbeat** | Schedule (hourly) | Autonomous task processing |

### Core Agent — n8nClaw
- **Model**: Claude Sonnet 4.5 (via OpenRouter)
- **Memory**: Postgres chat history (15-message context window)
- **Personality**: Configurable via "soul" field (name, vibe, purpose)
- **User Profile**: Living document updated as the agent learns about you

### Media Handling (Telegram)
| Media Type | Processing |
|-----------|-----------|
| Voice messages | Gemini 2.5 Flash transcription |
| Images | Gemini nano-banana-pro-preview analysis |
| Documents | Gemini 2.5 Flash document analysis |

### Tools
| Tool | Purpose |
|------|---------|
| **Get/Upsert Tasks** | Task management (n8n data tables) |
| **Get/Upsert Subtasks** | Subtask tracking linked by parent_task_id |
| **Update User & Heartbeat** | Persist user profile and heartbeat state |
| **Supabase Vector Store** | RAG — query past conversations for context |

### Sub-Agents
| Agent | Model | Purpose |
|-------|-------|---------|
| **Research Agent** | Gemini 3 Flash | Web research via Tavily + Wikipedia |
| **Email Manager** | Claude Haiku 4.5 | Gmail CRUD (read, reply, send, delete, search) |
| **Document Manager** | Claude Haiku 4.5 | Google Docs/Drive (create, update, move, delete) |
| **Worker 1** | Claude Haiku 4.5 | Simple tasks |
| **Worker 2** | Claude Sonnet 4.5 | Mid-level work |
| **Worker 3** | Claude Opus 4.6 | Higher-order thinking |

### Long-Term Memory Pipeline
A scheduled workflow that:
1. Pulls new chat history from Postgres
2. Aggregates and summarizes conversations (Haiku 4.5)
3. Embeds summaries (OpenAI embeddings)
4. Stores in Supabase vector database for RAG retrieval

## Setup

### Prerequisites
- [n8n](https://n8n.io) (self-hosted or cloud)
- Accounts/API keys for the services you want to use

### 1. Import the Workflow
1. Open n8n
2. Go to **Workflows** > **Import from File**
3. Select `n8nClaw.json`

### 2. Create Data Tables
Create three n8n data tables:

**Init Table** (user profile):
| Column | Type |
|--------|------|
| username | string |
| soul | string |
| user | string |
| heartbeat | string |
| last_channel | string |
| last_vector_id | number |

**Tasks Table**:
| Column | Type |
|--------|------|
| task_name | string |
| task_details | string |
| task_complete | boolean |
| Is_recurring | boolean |

**Subtasks Table**:
| Column | Type |
|--------|------|
| parent_task_id | string |
| subtask_name | string |
| subtask_details | string |
| subtask_complete | boolean |

### 3. Configure Credentials
Set up the following credentials in n8n (only configure what you need):

| Credential | Required For | Where to Get |
|-----------|-------------|-------------|
| **Telegram Bot API** | Telegram channel | [@BotFather](https://t.me/BotFather) |
| **OpenRouter API** | All AI models | [openrouter.ai](https://openrouter.ai) |
| **Postgres** | Chat memory | Your Postgres instance |
| **Supabase** | Vector store / RAG | [supabase.com](https://supabase.com) |
| **OpenAI API** | Embeddings | [platform.openai.com](https://platform.openai.com) |
| **Gmail OAuth2** | Email management | Google Cloud Console |
| **Evolution API** | WhatsApp | [evolution-api.com](https://evolution-api.com) |
| **Google AI (Gemini)** | Media processing | [ai.google.dev](https://ai.google.dev) |
| **Google Docs/Drive OAuth2** | Document management | Google Cloud Console |
| **Tavily API** | Web search | [tavily.com](https://tavily.com) |

### 4. Update Placeholders
Search the workflow JSON for `YOUR_` and replace with your actual values:

| Placeholder | What to Replace With |
|------------|---------------------|
| `YOUR_USERNAME` | Your chosen username |
| `YOUR_TELEGRAM_CHAT_ID` | Your Telegram chat ID (get it from [@userinfobot](https://t.me/userinfobot)) |
| `YOUR_PHONE` | Your phone number (for WhatsApp filtering) |
| `YOUR_EVOLUTION_INSTANCE` | Your Evolution API instance name |
| `YOUR_WEBHOOK_PATH` | Auto-generated on import (or set your own) |
| `YOUR_*_TABLE_ID` | The IDs of the data tables you created in step 2 |
| `YOUR_*_CREDENTIAL_ID` | Auto-populated when you connect credentials in n8n |
| `YOUR_PROJECT_ID` | Your n8n project ID |

### 5. Set Up Supabase Vector Store
Create a `documents` table in Supabase with the pgvector extension enabled. The table should match the schema expected by n8n's Supabase Vector Store node (with a `match_documents` function).

### 6. Activate
1. Connect all credentials in the n8n UI
2. Update the data table IDs to point to your tables
3. Update the filter nodes with your Telegram chat ID / WhatsApp number
4. Activate the workflow

## Customization

### Adding a New Channel
1. Add a new trigger node (webhook, poll, etc.)
2. Add a filter node to validate incoming messages
3. Add a "Get row(s)" node to fetch user profile from the Init table
4. Add an "Edit Fields" node to normalize into `user_message`, `system_prompt_details`, and `last_channel`
5. Connect to the n8nClaw agent
6. Add a new output in the Switch node to route responses back to the channel

### Changing Models
All models are configured via OpenRouter. To swap models, edit the model ID in any OpenRouter Chat Model node. The tiered worker system is:
- **Worker 1**: Cheap/fast model for simple tasks
- **Worker 2**: Mid-tier model for moderate complexity
- **Worker 3**: Best model for reasoning-heavy work

## License

MIT
