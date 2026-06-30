# Noga WhatsApp AI Assistant - System Design & Architecture

## 1. Overview & Objectives
This project details the architecture for **Noga**, a modular, Dockerized Home Assistant interface accessible via WhatsApp. It is powered by the **Google Gemini API (Google AI Studio)** and features dynamic model switching, bilingual support (Hebrew & English), and integration with home automation hubs (like Home Assistant Core).

Noga functions as an **Autonomous AI Agent** (inspired by Hermes and OpenClaw) that manages calendars, reads and writes its own Markdown-based knowledge base and skills repositories, and integrates with **Google Calendar** using a Google Service Account.

Additionally, Noga exposes a webhook gateway on port `3000` that handles proactive events (notifications, image uploads, and reminders) dispatched by Home Assistant automations, and exposes a connection status API back to Home Assistant.

To keep the deployment stack simple and consolidated, **Home Assistant communication is handled directly inside the core Orchestrator**. The orchestrator includes built-in clients to interact with HASS either via its standard REST API or by initiating direct Model Context Protocol (MCP) client connections, eliminating the need for separate connector images in the Docker stack.

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
  * Detects when the bot account joins a new group (using Baileys events like `chats.upsert` or `groups.update`).
  * Dispatches a registration hook to the Orchestrator with the new group JID and metadata.
  * Listens to incoming message reactions (e.g., `messages.reaction` event) and forwards them to the Orchestrator along with the sender, the reaction emoji, and the original message ID.

### 3.2 Web Admin Portal / Gateway Container (`admin-portal`)
* **Role:** Centralized configuration panel and entry API endpoint. Mapped to external host port `3000`.

### 3.3 Noga Core Orchestrator Container (`ai-orchestrator`)
* **Role:** The **Autonomous Agent Engine**, Webhook Server, and Home Assistant Driver.
* **Responsibilities:**
  * **Tenant Context Router:** Scopes variables, HASS configurations, and tool lists per tenant chat. If a message comes from a disabled/unapproved group chat, it is instantly discarded.
  * **Reaction Handler Router:** Receives reaction events $\rightarrow$ Resolves target tenant $\rightarrow$ Spawns the autonomous ReAct loop using the instructions written in the tenant's `/skills/{tenant_id}/reminders.md` file.
  * **Proactive Webhook API Router:** Exposes endpoints to receive HASS notifications and status queries. Authenticates requests using the tenant-specific `x-webhook-secret` header.
  * **Integrated Home Assistant Drivers:** REST API and MCP.
  * **Token Logger Middleware:** Records usage metadata from Gemini.
  * **Background Job & Scheduler Daemon:** Manages dynamic cron execution and schedules proactive message nudges.

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

#### Tenant Profile Schema (`tenants` table)
```json
{
  "tenant_id": "string (Primary Key)",
  "name": "string (e.g., 'Levi Home')",
  "enabled": "boolean",
  "reply_method": "ALL_MESSAGES | TAGGED | KEYWORD",
  "trigger_keywords": ["string"],
  "active_model": "string",
  "language": "string (he | en)",
  "enabled_tools": ["string"],
  "webhook_secret": "string (Tenant-specific x-webhook-secret check)",
  "hass_url": "string (Tenant-specific HASS base URL)",
  "hass_token": "string (Tenant-specific HASS long-lived token)",
  "google_calendar_id": "string (Tenant-specific Google Calendar ID)"
}
```

---

## 5. Dynamic Agentic Cronjobs Execution Lifecycle
The cron scheduler initiates background runs of the agent.

---

## 6. Proactive Webhook Gateway (Home Assistant Integration Specs)
Noga exposes REST endpoints on port `3000` to allow Home Assistant to broadcast notifications, schedule reminders, and query WhatsApp connection status.

Because each tenant operates independently, **webhooks are routed contextually based on the `x-webhook-secret` header**. When a request hits `/api/notify` or `/api/webhook/reminder`, Noga checks the secret header against the database tenant profiles, resolves which tenant owns that secret, and posts the notification to the correct WhatsApp group mapped to that tenant.

---

## 7. Modular Core Skills Catalog
Core capabilities are packaged as separate, loadable modules. Rather than hardcoding behavioral rules in the codebase, Noga's logic (like handling confirmations or retry schedules) is written entirely inside the skill `.md` files.

### 7.1 Reminders, Reactions & Nudging Skill (`/skills/{tenant_id}/reminders.md`)
* **Objective:** Manage dynamic alerts and allow users to mark reminders as done via chat text replies or WhatsApp **emoji reactions** (e.g., 👍, ❤️, "Like").
* **Behavior Definition (Non-Hardcoded):** The core code only exposes simple primitive database and scheduling tools. The rules governing emoji detection, completion triggers, check-back delays, and nudge schedules are described in the `/skills/{tenant_id}/reminders.md` Markdown instruction file.
* **Orchestrator Primitive Tools:**
  * `get_reminder_by_msg_id(message_id)`: Fetches a reminder record linked to the specific WhatsApp message.
  * `update_reminder_status(reminder_id, status)`: Marks reminder as `done` or `active`.
  * `schedule_nudge(reminder_id, minutes, count)`: Registers a delayed execution trigger to run the check-in script after $X$ minutes.
  * `cancel_nudge(reminder_id)`: Deletes any pending scheduled nudge jobs.
  * `get_nudge_config()`: Fetches the tenant's configured nudge parameters: $X$ (minutes interval) and $N$ (maximum retry count).

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
     - Immediately call `cancel_nudge(reminder_id)` to stop recurring notifications.
     - Respond to the group confirming completion (e.g., "תודה! סומן שבוצע" or "Reminder marked as done!").

## 2. Handling Text Confirmation
* **Trigger:** ON_GROUP_MESSAGE (Reply to reminder message)
* **Instructions:**
  1. If a user replies directly to a reminder message with text like "done", "בוצע", "עשיתי", or "טופל":
     - Resolve the reminder using the parent message ID.
     - Call `update_reminder_status(reminder_id, "done")`.
     - Call `cancel_nudge(reminder_id)`.

## 3. Nudging & Retry Schedule
* **Trigger:** ON_REMINDER_FIRED
* **Instructions:**
  1. When a reminder is broadcast, create a record in the database linking the sent `message_id` with the reminder.
  2. Read nudge configs using `get_nudge_config()` to fetch the interval ($X$ minutes) and max retries ($N$).
  3. Call `schedule_nudge(reminder_id, X, 1)` to schedule the first check-in.

* **Trigger:** ON_NUDGE_TRIGGERED
* **Context variables:** `reminder_id`, `current_retry_count`
* **Instructions:**
  1. Query the reminder status. If status is "done", exit silently.
  2. If status is still "active":
     - Fetch the max retries limit ($N$) from `get_nudge_config()`.
     - If `current_retry_count` < N:
       - Resend the reminder text as a nudge to the group (e.g. "Reminder nudge: Dryer has finished! Can someone please empty it?").
       - Increment `current_retry_count`.
       - Call `schedule_nudge(reminder_id, X, current_retry_count)` to schedule the next check-in.
     - If `current_retry_count` >= N:
       - Send a final notice ("Max retries reached. Reminder ignored.") and stop scheduling checks.
```

### 7.2 Shopping List
Confined to `/knowledge/{tenant_id}/shopping_list.md`.

### 7.3 Web Browsing & Information Retrieval
General tool, enabled/disabled per tenant profile.

### 7.4 Home Assistant Automation
REST and MCP drivers handled directly inside the orchestrator container.

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
