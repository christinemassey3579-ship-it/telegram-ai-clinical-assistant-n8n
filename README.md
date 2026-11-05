# Telegram-based AI Clinical Assistant (n8n · RAG + MCP)

A Telegram bot for clinic staff, built in **n8n**. Staff message the bot in plain language. It answers medicine-dosage and guideline questions from a clinic knowledge base (**RAG**), and it carries out actions through an **MCP server**: booking follow-ups in Google Calendar, adding tasks, and emailing patient summaries to doctors.

**Built by:** Christine Massey · **Tools:** n8n, Telegram, Claude (Anthropic), Google Gemini embeddings, Google Calendar, Gmail, n8n Data Tables

> Demonstration project only. The clinical guidelines are sample data and are not medical advice.

## What it can do

| Staff types in Telegram | What the assistant does |
| --- | --- |
| "What's the starting dose of metformin?" | Searches the knowledge base and answers from it only, naming the guideline section |
| "Schedule a follow-up for Ravi Kumar next Tuesday at 10am" | Creates a Google Calendar event (30 min, or 45 for a new patient) |
| "Create a task to upload lab results for Priya" | Adds a row to the Clinical Tasks table |
| "Email Dr Sharma a summary for Ravi Kumar" | Writes a summary in the clinic format and sends it by Gmail |
| A message mentioning chest pain or stroke signs | Tells staff to seek emergency care (112 / 108) and inform the doctor |

## How it works

The project is **two n8n workflows** that work together:

**1. Telegram Bot (RAG + MCP)**, in `clinical-assistant-telegram-bot.workflow.json`:

```
Telegram message → Clinical Assistant Agent (Claude) → Reply in Telegram
                         ├── Chat Memory (last 10 messages per chat)
                         ├── clinical_knowledge_base (RAG search, Gemini embeddings)
                         └── Clinical Actions (MCP client) → workflow 2
```

- **Part A, the assistant:** the AI agent decides whether to search the knowledge base, call an action, or ask a follow-up question.
- **Part B, loading the knowledge base:** a one-click branch turns the 12 guideline sections into Gemini embeddings and stores them in n8n's in-memory vector store under the key `clinical_kb`.

**2. Actions (MCP Server)**, in `clinical-assistant-actions-mcp-server.workflow.json`, offers three tools to the assistant:

| Tool | Connects to | What it does |
| --- | --- | --- |
| `schedule_patient_followup` | Google Calendar | Creates a follow-up appointment (India time, +05:30) |
| `create_clinical_task` | n8n Data Table "Clinical Tasks" | Adds a task with priority, due date and status |
| `email_doctor_summary` | Gmail | Sends a plain-text patient summary to a doctor |

## Safety rules built into the assistant

- Dosage answers come only from the knowledge base. It never invents doses.
- Every dosage answer ends with "Please confirm with the prescribing doctor."
- It asks for missing details (patient name, date, doctor's email) instead of guessing.
- It recognises red-flag symptoms and gives emergency guidance.

## Files

| File | What it is |
| --- | --- |
| `clinical-assistant-telegram-bot.workflow.json` | Workflow 1: the Telegram bot, AI agent, RAG knowledge base and loader |
| `clinical-assistant-actions-mcp-server.workflow.json` | Workflow 2: the MCP server with the three action tools |
| `clinical-guidelines-knowledge-base.md` | The 12 sample guideline sections loaded into the knowledge base, for easy reading |
| `README.md` | This description |

## How to set it up

1. **Import both workflows** into n8n with **⋯ → Import from file**.
2. **Connect your own accounts**: Telegram (bot token from @BotFather), Anthropic, Google Gemini, Google Calendar and Gmail. No keys or account IDs are stored here; they appear as `YOUR_..._ID` placeholders.
3. **Create a Data Table** called "Clinical Tasks" with the columns task, patient_name, priority, due_date, status and notes. Then select it in `create_clinical_task`.
4. **Pick your calendar** in `schedule_patient_followup`.
5. **Publish the MCP Server workflow**, copy its production URL, and paste it into the **Clinical Actions (MCP)** node of the bot workflow, replacing `https://YOUR-N8N-INSTANCE/mcp/clinical-actions`.
6. **Recommended:** turn on authentication (for example a Bearer token) on the MCP Server trigger and add the same token to the MCP client node, so only your bot can use the action tools.
7. In the bot workflow, click **Load Knowledge Base (click to run)** once.
8. **Publish the bot workflow** and send your bot a message on Telegram.
