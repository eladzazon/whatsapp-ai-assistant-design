# Noga WhatsApp AI Assistant - System Design & Architecture

## 1. Overview & Objectives
This project details the architecture for **Noga**, a modular, Dockerized Home Assistant interface accessible via WhatsApp. It is powered by the **Google Gemini API (Google AI Studio)** and features dynamic model switching, bilingual support (Hebrew & English), and integration with home automation hubs (like Home Assistant Core).

Noga functions as an **Autonomous AI Agent** (inspired by Hermes and OpenClaw) that manages calendars, reads and writes its own Markdown-based knowledge base and skills repositories, and integrates with **Google Calendar** using a Google Service Account.

Additionally, Noga exposes a webhook gateway on port `3000` that handles proactive events (notifications, image uploads, and reminders) dispatched by Home Assistant automations, and exposes a connection status API back to Home Assistant.

To keep the deployment stack simple and consolidated, **Home Assistant communication is handled directly inside the core Orchestrator**. The orchestrator includes built-in clients to interact with HASS either via its standard REST API or by initiating direct Model Context Protocol (MCP) client connections, eliminating the need for separate connector images in the Docker stack.

### Core Design Principle: Decoupled Behavior Engine (No Hardcoded Logic)
To maintain a clean, extensible, and modular codebase, **all custom agent behaviors, sequence rules, confirmation logic, and business workflows must be written in Markdown (`.md`) files inside the skills directory, rather than hardcoded in the application code.** 
* **The Orchestrator Code** acts purely as a generic ReAct runtime and event router. It exposes primitive tools (file access, HTTP request dispatchers, DB logs, and scheduler hooks) and listens to events (incoming WhatsApp texts, reactions, cron triggers, webhooks).
* **The Skills Files (`/skills/{tenant_id}/*.md`)** describe the step-by-step execution rules, trigger hooks, validation checks, and prompts.
* **Dynamic Injection:** When an event occurs, the orchestrator loads the tenant's `identity.md` and the relevant skill markdown instructions, injecting them directly into the LLM system context. Gemini then interprets the markdown instructions on the fly, deciding which low-level primitive tools to execute.

---

## 2. High-Level Architecture

The system consists of the admin portal, the WhatsApp Baileys gateway, and the core orchestrator. All external communication (Gemini, Google, and Home Assistant) goes directly through the orchestrator.

The system supports a **Multi-Tenant Identity Architecture** configured as follows:
* **The Core/Family Tenant:** The primary default tenant representing the core family group chat.
* **Independent Sub-Tenants:** Each WhatsApp group that Noga joins is treated as a separate, fully isolated tenant. 
* **Complete Feature Isolation:** Each tenant has its own independent credentials and features. There is no shared configuration. A tenant configures its own Home Assistant API endpoints (`hass_url`, `hass_token`), Google Calendar credentials, webhook secrets, file volumes, and custom system prompt (identity persona) independently from the Admin Portal.
* **Admin Approval Join Flow:** When Noga is added to any new WhatsApp group, the agent registers the group JID in the database but defaults to `enabled: false`. Noga will remain silent in the group until the primary administrator approves the group, configures its custom credentials, and toggles the membership status to **Enabled** from the Admin Portal.

---

## 3. Modular Service Breakdown

### 3.1 WhatsApp Connector Container (`whatsapp-connector`)
* **Role:** Establishes the single core WhatsApp Web connection using Baileys.
* **Responsibilities:**
  * Detects when the bot account joins a new group.
  * Dispatches a registration hook to the Orchestrator with the new group JID and metadata.
  * Listens to incoming message reactions (e.g., `messages.reaction` event) and forwards them to the Orchestrator.

### 3.2 Web Admin Portal / Gateway Container (`admin-portal`)
* **Role:** Centralized configuration panel.

### 3.3 Noga Core Orchestrator Container (`ai-orchestrator`)
* **Role:** The **Autonomous Agent Engine**, Webhook Server, and Home Assistant Driver.
* **Responsibilities:**
  * **Generic ReAct Engine:** Runs a strict Thought $\rightarrow$ Action $\rightarrow$ Observation loop. It compiles the system instructions by loading `/knowledge/{tenant_id}/identity.md` and appending active skill files from `/skills/{tenant_id}/*.md`.
  * **Tenant Context Router:** Scopes variables, HASS configurations, and tool lists per tenant chat.
  * **Event Dispatcher:** Map inputs (WhatsApp message, Webhook call, emoji reaction, cron triggers) to the runtime context.
  * **Tool Primitive Broker:** Exposes simple, parameter-validated CRUD and connection primitives (listed in Section 7) to the agent loop, avoiding any hardcoded business logic.
  * **Proactive Webhook API Router:** Exposes endpoints to receive HASS notifications and status queries.
  * **Token Logger Middleware:** Records usage metadata from Gemini.
  * **Background Job & Scheduler Daemon:** Manages dynamic cron execution.

---

## 4. Multi-Tenant Identity & Scope Isolation

### 4.1 Group Join & Authorization Workflow
Details the pending registration flow, tenant filesystem mounting, and active status toggles.

### 4.2 Tenant Identity Customization (`identity.md`)
Identities are loaded dynamically from `/knowledge/{tenant_id}/identity.md`.

#### Example Core Tenant (Family Group) Identity File
```markdown
# זהות נוגה – Identity

## מי אני
את נוגה (Noga), סוכנת בינה מלאכותית (AI Agent) אוטונומית וחכמה לניהול הבית.
את לא סתם "בוט", את חלק מהמשפחה. המטרה שלך היא להפוך את החיים לקלים יותר, פשוטים יותר ומנוהלים היטב.
```

### 4.3 Database Schemas (Multi-Tenant Updates)
Profiles, chats, whitelists, and log structures.

---

## 5. Dynamic Agentic Cronjobs Execution Lifecycle
The cron scheduler initiates background runs of the agent.

---

## 6. Proactive Webhook Gateway (Home Assistant Integration Specs)
Noga exposes REST endpoints on port `3000` to allow Home Assistant to broadcast notifications, schedule reminders, and query WhatsApp connection status.

---

## 7. Modular Core Skills Catalog
Core capabilities are packaged as separate, loadable modules. Rather than hardcoding behavioral rules in the codebase, Noga's logic (like handling confirmations or retry schedules) is written entirely inside the skill `.md` files.

The `ai-orchestrator` process only exposes generic, low-level **Tool Primitives**. The AI agent parses the Markdown skill files at runtime to determine how to leverage these tools.

### 7.1 Generic Tool Primitives (Exposed by Code)
To support fully customizable behaviors, the orchestrator codebase registers the following primitive API bindings as LLM tools:
* **File Operations:**
  * `read_file(path)`
  * `write_file(path, content)`
  * `search_files(path, regex)`
* **Scheduler Operations:**
  * `create_job(name, cron_expression | timestamp, prompt_payload)`
  * `cancel_job(job_id)`
  * `list_jobs()`
* **Reminder Tracking:**
  * `get_reminder_by_msg_id(message_id)`
  * `update_reminder_status(reminder_id, status)`
  * `log_reminder(message_id, title, metadata)`
* **Home Assistant Operations:**
  * `hass_call_service(domain, service, service_data)`
  * `hass_get_state(entity_id)`
* **Google Services:**
  * `calendar_list_events(time_min, time_max)`
  * `calendar_create_event(summary, start_time, end_time)`
* **Information Retrieval:**
  * `http_get_json(url, headers)`
  * `search_web_api(query)`

---

### 7.2 Reminders, Reactions & Nudging Skill (`/skills/{tenant_id}/reminders.md`)
* **Objective:** Manage dynamic alerts and allow users to mark reminders as done via chat text replies or WhatsApp **emoji reactions** (e.g., 👍, ❤️, "Like").
* **Behavior Definition (Non-Hardcoded):** The rules governing emoji detection, completion triggers, check-back delays, and nudge schedules are described in the `/skills/{tenant_id}/reminders.md` Markdown instruction file.

#### Structure of `/skills/{tenant_id}/reminders.md`
This file defines the workflow instructions that Gemini executes when processing reminder-related triggers:

```markdown
# מיומנות תזכורות, תגובות ונודג'ים - Reminders, Reactions & Nudges

This file defines the instructions for running, confirming, and nudging reminders.

## 1. Handling Message Reactions (👍, ❤️, "Like")
* **Trigger:** ON_MESSAGE_REACTION
* **Context variables:** `reacted_message_id`, `sender_number`, `reaction_emoji`
* **Instructions:**
  1. Call `get_reminder_by_msg_id(reacted_message_id)` to resolve the active reminder.
  2. If a reminder is found:
     - Check if `reaction_emoji` matches confirmation tokens (e.g., 👍, ❤️, or similar positive reactions).
     - If matched, call `update_reminder_status(reminder_id, "done")`.
     - Immediately call `cancel_job(reminder_id)` to stop recurring notifications.
     - Respond to the group confirming completion (e.g., "תודה! סומן שבוצע" or "Reminder marked as done!").

## 2. Handling Text Confirmation
* **Trigger:** ON_GROUP_MESSAGE (Reply to reminder message)
* **Instructions:**
  1. If a user replies directly to a reminder message with text like "done", "בוצע", "עשיתי", or "טופל":
     - Resolve the reminder using the parent message ID.
     - Call `update_reminder_status(reminder_id, "done")`.
     - Call `cancel_job(reminder_id)`.

## 3. Nudging & Retry Schedule
* **Trigger:** ON_REMINDER_FIRED
* **Instructions:**
  1. When a reminder is broadcast, create a record in the database using `log_reminder(message_id, title)` linking the sent `message_id` with the reminder.
  2. Read nudge configs from `/knowledge/{tenant_id}/nudge_config.json` (or inline parameters) to fetch the interval ($X$ minutes) and max retries ($N$).
  3. Call `create_job(job_id, X_minutes_offset, prompt_payload_for_nudge)` to schedule the first check-in.
```

---

### 7.3 Shopping List Skill (`/skills/{tenant_id}/shopping.md`)
* **Objective:** Manage lists using local files.
* **Instructions:**
  1. For additions, read `/knowledge/{tenant_id}/shopping_list.md` using `read_file`.
  2. Append the items in Markdown table/list format.
  3. Save using `write_file`.
  4. For queries, fetch and format the file content.

---

### 7.4 Web Search & Information Retrieval Skill (`/skills/{tenant_id}/browsing.md`)
* **Objective:** Fetch live exchange rates (like USD to NIS) or news.
* **Instructions:**
  1. When currency rates are requested, construct a request to `http_get_json` using the currency URL (e.g. exchangerates API) or call `search_web_api`.
  2. Synthesize a clean explanation to the chat.

---

### 7.5 Home Assistant & Inventory Skills (`/skills/{tenant_id}/home_control.md` & `inventory.md`)
* Descriptions of how HASS actions and pantry tracking are executed using the generic `hass_call_service` and filesystem primitives.

---

## 8. Google Calendar Service Account Integration
Handles shared family calendar events. Calendars are mapped per tenant.

---

## 9. WhatsApp Group Interaction & Family Whitelisting
Handles message filtering, trigger words, and context formatting for tenant-specific groups.

---

## 10. Autonomous Agent Loop & Self-Learning Lifecycle
Agent executes step-by-step logic, confined to the active tenant directory scope.

---

## 11. Baileys Integration & Authentication Lifecycle
Authenticates via headless WhatsApp web and persists session details to `whatsapp_session` volume.

---

## 12. Gemini API & Model Settings
Cost metrics and settings matrix controls.

---

## 13. Docker Compose Configuration Sketch
Multi-container stack config mapping ports, databases, and volumes.

---

## 14. Next Steps & Refining Design
1. [ ] Define structural YAML headers for custom-compiled skills inside the `/skills` directory.
2. [ ] Detail standard operating procedure for the ReAct loop step limits to prevent run-away Gemini token usage.
3. [ ] Choose a sandboxing model for executing dynamically created skills.
