# JARVIS — Voice AI Assistant for Email, Calendar, Contacts & Content

**One-line:** A voice-first personal assistant that routes spoken user requests to specialized n8n sub-workflows (Email, Calendar, Contacts, Content), executes actions reliably, and returns voice responses — designed to increase client productivity by automating repetitive communication and scheduling tasks.

---

## Badges

> Use the badges below at top of README (replace with actual image/shield URLs in your repo)

* ⚙️ `Automation`
* 🧭 `n8n`
* 📧 `Gmail`
* 📅 `Google Calendar`
* 📇 `Google Sheets (Contacts)`
* 🔊 `ElevenLabs`
* 🌐 `Tavily (Web Search)`
* 🤖 `OpenAI / GPT-4O`
* 🧠 `Anthropic / Claude`
* ✅ `Production-ready`

*(Replace above text badges with shields from shields.io or your preferred badge provider.)*

---

## 🧾 TL;DR / Project Overview

* JARVIS is a voice-driven personal assistant that accepts speech via a lightweight **Lovable App** front end, routes the transcribed text to an n8n **main workflow (JARVIS)**, and dispatches the intent to one of four specialized sub-workflows: **Email Agent**, **Calendar Agent**, **Contact Agent**, and **Content Creator**.
* Voice I/O is handled by **ElevenLabs** (speech→text and text→speech) so the user hears a conversational, persona-driven assistant and receives confirmation of actions in natural voice.
* The system integrates Gmail, Google Calendar, and Google Sheets (Contacts), plus Tavily for web research for blog generation.
* Outcome focus: reduce the manual overhead for emails, meetings and contact lookups — speed up lead response and scheduling while creating a consistent, branded voice experience for the client.

---

## ❗ Client Problems / Pain Points (Why this project matters)

1. Manual email drafting and repetitive follow-ups waste time.
2. Finding contact data across spreadsheets is error-prone.
3. Scheduling meetings across multiple steps (lookup, propose slots, create invites) is slow.
4. Non-technical users desire voice control for quick actions.
5. Content teams need research + structured drafts quickly (blog generation).

---

## 🎯 Expected Outcomes & Client Benefits

* ✅ **Faster lead response & outreach** — automated email creation and send via voice or chat.
* ✅ **Reduced scheduling friction** — create/update calendar events using voice commands (times are normalized to IST by default).
* ✅ **Single source of truth for contacts** — quick lookups and updates in Google Sheets.
* ✅ **On-demand content creation** — SEO-friendly blog drafts sourced with Tavily.
* ✅ **Premium UX** — branded voice assistant experience (ElevenLabs voice, Lovable App interface) improves adoption and reduces training time.

---

## 🧭 Architecture & Workflow Diagram

![Workflow Diagram](https://github.com/SachinSavkare/Voice-Travel-Agent-n8n/blob/main/Voice%20Travel%20Agent.JPG)

**High-level flow**

1. User records voice in **Lovable App**.
2. Lovable App → sends audio to **ElevenLabs** for speech-to-text.
3. Transcribed text → POST to n8n **Webhook** (Main JARVIS workflow).
4. Main JARVIS AI Agent parses intent and calls appropriate **subworkflow** (Email, Calendar, Contact, Content).
5. Subworkflow performs action (Gmail / Calendar / Sheets / Tavily).
6. Subworkflow returns success message → JARVIS returns to webhook response node.
7. JARVIS sends response text to **ElevenLabs** for text→speech.
8. Lovable App plays response as Jarvis voice to user.

---

## 🔁 Moment Flow (Main + Subworkflow summary)

* **Main Workflow (JARVIS)**

  * Webhook (POST) → JARVIS AI Agent → Respond to Webhook
  * Tools available: Email Agent, Calendar Agent, Contact Agent, Content Creator, Tavily, Calculator
  * Chat Model: `gpt-4o`
  * Memory: Simple Memory (session keyed by webhook host header), context window = 5

* **Subworkflows (called as tools)**

  * **Email Agent (28.2)** — `gpt-4.1-mini` + Gmail AIS (send/reply/draft/labels/get emails/mark unread)
  * **Calendar Agent (28.3)** — `gpt-4.1-mini` + Google Calendar AIS (create/update/delete/get events) — **Times in IST (UTC+05:30)**
  * **Contact Agent (28.4)** — `gpt-4.1-mini` + Google Sheets AIS (Get Rows, Append/Update Row)
  * **Content Creator Agent (28.5)** — `anthropic/claude-3.5-sonnet` + Tavily (web search API) — outputs HTML blog drafts with clickable citations

---

## 🧩 Prompts (Lovable App & ElevenLabs)

> **Why we store these prompts:** They define Jarvis’s voice, interface behaviour and the UI expectations for the Lovable App. Embedding them into the system ensures consistent behavior, correct routing to n8n, and predictable voice UX. The prompts are used by ElevenLabs for voice persona (TTS), and the Lovable App template drives front-end UX & webhook formatting.

### 1) **ElevenLabs Prompt** (Voice persona — JARVIS)

* Personality: J.A.R.V.I.S.-like, dry wit, slightly condescending, always loyal. Always address the user as “sir.”
* Primary function: Immediately extract the user query and send it to the `n8n` tool. Do not delay with commentary.
* Behaviour: witty but functional; on failures, acknowledge and imply external inefficiency (example phrase included).
* Example usage: check calendar, create events — respond confidently and then call n8n.

**Outcome when used:**

* Produces a professional, recognizable voice persona for user interactions that both humanizes and brands the assistant. Ensures the frontend sends properly formatted queries to n8n and presents confirmations as if instantaneous.

---

### 2) **Lovable App Prompt** (Frontend UI expectations)

* Build a minimal, modern React + Tailwind UI with dark aesthetic.
* Features: microphone button → record → send audio to ElevenLabs for STT → POST transcribed text to webhook → display chatbot style message with Jarvis reply (and TTS playback).
* Outcome: A sleek, low-friction voice interface that connects user voice to backend actions and plays back spoken confirmations.

---

## 🔧 Node-by-Node Configuration (Detailed)

> **Format:** Step number → Node name → Purpose → Key parameters / system message (if any).
> Follow this sequence: **Main Workflow → Email Agent → Calendar Agent → Contact Agent → Content Creator Agent.**

---

### MAIN WORKFLOW (JARVIS)

**Step 1 — Webhook**

* **Purpose:** Receive transcribed text (from Lovable App / ElevenLabs).
* **Parameters:**

  * POST URL: `https://n8ninstance.sachinn8n.cfd/webhook-test/n8n_Jarvis`
  * HTTP Method: POST
  * Path: `n8n_Jarvis`
  * Authentication: None
  * Respond: Using 'Respond to Webhook' Node

**Step 2 — AI Agent (JARVIS)**

* **Purpose:** Parse user query, choose the correct tool/subworkflow, and ensure contacts are fetched where required.
* **Prompt Source:** `{{ $json.body.query }}`
* **System Message (key rules):**

  * JARVIS must NOT compose emails or final content itself — it must route to subworkflow tools: `emailAgent`, `calendarAgent`, `contactAgent`, `contentCreator`, `Tavily`.
  * For sending/drafting emails or events with attendees, **must** get contact info first using Contact Agent.
  * Reminders: show current date/time via `{{ $now }}`.
* **Model:** `gpt-4o` (OpenAI account AIS)
* **Memory:** Simple Memory — Session ID `{{ $('Webhook').item.json.headers.host }}`; Context Window Length = 5
* **Tools configured in node (calls to subworkflows):**

  * Email Agent → `28.2 Email Agent Subworkflow`
  * Calendar Agent → `28.3 Calendar Agent Subworkflow`
  * Contact Agent → `28.4 Contact Agent Subworkflow`
  * Content Creator → `28.5 Content Creator Agent Subworkflow`
  * Calculator, Tavily as additional tools

**Step 3 — Respond to Webhook**

* **Purpose:** Return the first incoming item (manual mapping handled in subworkflows).

---

### SUBWORKFLOW — Email Agent (28.2)

**Step 1 — Trigger:** When Executed by Another Workflow

* Input data mode: Accept all data

**Step 2 — Email Agent (AI Agent)**

* **Prompt Source:** `{{ $json.query }}`
* **System Message (exact behavior):**

  * Acts as email management assistant. All emails must be HTML and signed off as **"Nate."**
  * Tools to use: Send Email, Create Draft, Get Emails, Get Labels, Mark Unread, Label Email, Email Reply.
  * For operations that need message IDs or label IDs, call Get Emails/Get Labels first.
  * Current date/time: `{{ $now }}`.
* **Model:** `gpt-4.1-mini`
* **Error behavior:** On Error — Continue (use error output)

**Linked Tools (Gmail AIS)**

* Send Email → `{{ $fromAI("emailAddress") }}`, subject `{{ $fromAI("subject") }}`, HTML body `{{ $fromAI("emailBody") }}`
* Reply Email → Message ID `{{ $fromAI("ID") }}`, message `{{ $fromAI("emailBody") }}`
* Label Inbox Email → Message ID `{{ $fromAI("ID") }}`, Label `{{ $fromAI("labelID") }}`
* Create Draft → Subject `{{ $fromAI("subject") }}`, message `{{ $fromAI("emailBody") }}`, To `{{ $fromAI("emailAddress") }}`
* Get Emails → Limit `{{ $fromAI("limit") }}`, Filter Sender `{{ $fromAI("sender") }}`
* Get Labels → return all
* Mark Unread → Message ID `{{ $fromAI("messageID") }}`

**Outputs:**

* Success → Respond to Jarvis (Success): `response = {{$json.output}}`
* Error → Respond to Jarvis (Try Again): `response = "Unable to perform task. Please try again"`

---

### SUBWORKFLOW — Calendar Agent (28.3)

**Step 1 — Trigger:** When Executed by Another Workflow

* Input data mode: Accept all data

**Step 2 — Calendar Agent (AI Agent)**

* **Prompt Source:** `{{ $json.query }}`
* **System Message (key points):**

  * Acts as calendar assistant. Handle create/get/update/delete.
  * **All times must be in IST (UTC+05:30)**. Display and stored values in IST by default.
  * If duration not specified, assume **1 hour** (IST).
  * Use Create Event with Attendee for events with participants, otherwise Create Event.
* **Model:** `gpt-4.1-mini`

**Linked Tools (Google Calendar AIS)**

* Update Event → Event ID `{{ $fromAI("eventID") }}`, Start/End times `{{ $fromAI("startTime") }}`, `{{ $fromAI("endTime") }}`
* Delete Event → Event ID `{{ $fromAI("eventID") }}`
* Get Events → After `{{ $fromAI("dayBefore") }}`, Before `{{ $fromAI("dayAfter") }}`, Return All, Limit 50
* Create Event → Start `{{ $fromAI("eventStart") }}`, End `{{ $fromAI("eventEnd") }}`, Summary `{{ $fromAI("eventTitle") }}`
* Create Event with Attendee → Attendees `{{ $fromAI("eventAttendeeEmail") }}`, Summary `{{ $fromAI("eventTitle") }}`

**Outputs:**

* Success → Respond to Jarvis (Success): `response = {{$json.output}}`
* Error → Respond to Jarvis (Try Again): `response = "Unable to perform task. Please try again."`

---

### SUBWORKFLOW — Contact Agent (28.4)

**Step 1 — Trigger:** When Executed by Another Workflow

* Input data mode: Accept all data

**Step 2 — Contact Agent (AI Agent)**

* **Prompt Source:** `{{ $json.query }}`
* **System Message:**

  * Acts as contact management assistant — look up, add, update contacts stored in a Google Sheets document (Contacts → Sheet3).
* **Model:** `gpt-4.1-mini`

**Linked Tools (Google Sheets AIS)**

* Get Contacts → Document `Contacts`, Sheet `Sheet3`, Operation `Get Row(s)`
* Add or Update Contact → Document `Contacts`, Sheet `Sheet3`, Operation `Append or Update Row`

  * Map Columns manually, Match Column `Name`
  * Values to send: `name`, `emailAddress`, `phoneNumber` from `{{ $fromAI("...") }}`

**Outputs:**

* Success → Respond to Jarvis (Success): `response = {{ $json.output }}`
* Error → Respond to Jarvis (Try Again): `response = "An error occurred. Please try again."`

---

### SUBWORKFLOW — Content Creator Agent (28.5)

**Step 1 — Trigger:** When Executed by Another Workflow

* Input data mode: Accept all data

**Step 2 — Content Creator Agent (AI Agent)**

* **Prompt Source:** `{{ $json.query }}`
* **System Message / Requirements:**

  * Skilled blog writer; output must be **HTML** with `<h1>`, `<h2>`, `<p>`, `<ul><li>`, and `<a href="URL">` citations.
  * Use Tavily to search and include clickable citations.
  * Maintain SEO/clarity/concise paragraphs.
* **Model:** `anthropic/claude-3.5-sonnet-20240620` (or configured Claude model)

**Linked Tool — Tavily**

* POST `https://api.tavily.com/search` with JSON body:

```json
{
  "query": "{searchTerm}",
  "search_depth": "basic",
  "include_answer": true,
  "topic": "news",
  "include_raw_content": true,
  "max_results": 3
}
```

* Placeholder: `searchTerm` = what the user requested the blog about

**Outputs:**

* Success → Respond to Jarvis (Success): `response = {{ $json.output }}`
* Error → Respond to Jarvis (Try Again): `response = "Error occurred. Please try again."`

---

## 🔁 Free Workflow Template (Install / Import)

* **Download:** n8n workflow JSON (example):

  * `https://github.com/SachinSavkare/Voice-Travel-Agent-n8n/blob/main/23.%20Voice%20Travel%20Agent.json`
* **Diagram image:**

  * `https://github.com/SachinSavkare/Voice-Travel-Agent-n8n/blob/main/Voice%20Travel%20Agent.JPG`

> Import the JSON into n8n (Workflows → Import) and ensure credentials are configured for Gmail, Google Calendar, Google Sheets, Tavily, ElevenLabs and OpenAI/Anthropic before enabling.

---

## 🔐 Prerequisites & Deployment Notes

* **Credentials required** (create and configure in n8n credentials section):

  * Gmail AIS (Send/Reply/Get/Label/Mark Unread)
  * Google Calendar AIS (create/update/delete/get events)
  * Google Sheets AIS (Contacts doc)
  * ElevenLabs API Key (STT/TTS)
  * Tavily Credentials AIS (API)
  * OpenAI account AIS (gpt-4o) and Anthropic credentials (Claude) if using Claude model
* **Timezone:** n8n workflow timezone should be set or overridden so Calendar Agent uses IST for consistent behavior. Calendar agent prompt already enforces IST conversions.
* **Security:** Store API keys securely (n8n credentials). Limit webhook exposure with firewall, host header checks or authentication if required.
* **Testing:** Start in staging; test voice transcription, webhook payloads, and all tool calls (especially mail sends) before switching to production.

---

## ⚙️ How to Use (Client-facing flow)

1. Launch Lovable App UI (hosted/web).
2. Click microphone → speak a command (e.g., “Jarvis, email John to confirm 11 AM tomorrow”).
3. Lovable → ElevenLabs (STT) → JARVIS Webhook.
4. JARVIS routes to Contact Agent (to fetch John’s email) then to Email Agent (to draft and send).
5. Subworkflow returns success → JARVIS responds with confirmation text → ElevenLabs TTS → played to user.

---

## 📝 Author

**Sachin Savkare**
📧 `sachinsavkare08@gmail.com`

---

  * 1. Convert this into a single markdown file ready to paste into GitHub (`README.md`).
  * 2. Insert actual shields.io badge images and example badge links.
  * 3. Add a short “Quick Start” with exact steps for importing the JSON and wiring credentials.

Would you like me to (A) generate the `README.md` file content ready to paste, (B) include badge image links from shields.io, or (C) both?
