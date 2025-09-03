# Exno.3-Scenario-Based Report Development Utilizing Diverse Prompting Techniques
### DATE:29/08/2025                                                                 
### REGISTER NUMBER : 212222240036
### Aim: To design an AI-powered chatbot that assists customers in resolving issues related to product troubleshooting, order tracking, and general inquiries. The chatbot should handle various customer queries efficiently while maintaining a conversational and user-friendly tone. In this experiment, we will employ different prompt patterns to guide the development process of the chatbot, ranging from basic task-oriented prompts to more complex, persona-driven prompts.

### Algorithm:  # AI Customer Support Chatbot — Prompt Pattern Playbook & Design

**Aim:** Design an AI-powered chatbot that resolves product troubleshooting, order tracking, and general inquiries with a conversational, user‑friendly tone. This playbook provides ready‑to‑use prompt patterns, tool schemas, guardrails, and evaluation plans.

---

## 1) Scope & Objectives

* **Primary intents:**

  1. Troubleshooting (guide users through fixes; collect logs/context).
  2. Order tracking (ETAs, status, returns, cancellations).
  3. General inquiries (pricing, availability, warranty, policies, store hours).
* **Success metrics:** First Contact Resolution (FCR), avg. handle time (AHT), containment rate, CSAT, escalation rate, and policy adherence.
* **Channels:** Web widget, mobile app, WhatsApp/FB Messenger, email handoff.
* **Constraints:** No hallucinated facts; cite KB articles when used; escalate when confidence is low.

---

## 2) Core Intents, Entities & Slots

### Intents

* **TROUBLESHOOT** — Diagnose and fix product issues.
* **TRACK\_ORDER** — Retrieve order status/ETA, shipping, returns.
* **GENERAL\_INFO** — Answer FAQs about products, services, policies.
* **ESCALATE** — Handoff to human support.
* **SMALL\_TALK** — Greetings, chitchat; keep brief and helpful.

### Key Entities & Slots

* User identifiers: `email`, `phone`, `customer_id` (auth gated)
* Order fields: `order_id`, `tracking_number`, `carrier`
* Product fields: `product_id`, `product_name`, `model`, `serial_number`
* Troubleshooting fields: `issue_symptom`, `error_code`, `os_version`, `steps_tried`
* Logistics: `address_city`, `pincode`, `preferred_time_window`

**Slot policy:** Minimal ask-first, progressive disclosure (ask only what’s missing), confirm before actions ("Just to confirm…").

---

## 3) Knowledge & Tools

* **RAG sources:** FAQ pages, product manuals, SOPs, policy docs, outage bulletin.
* **APIs/Tools (function-calling):**

  * `get_order_status(order_id | email, phone)` -> status, ETA, items, tracking.
  * `get_tracking_details(tracking_number)` -> latest scan + location.
  * `get_product_guide(product_id|model)` -> steps, diagrams, warranty.
  * `suggest_troubleshoot_steps(model, issue_symptom, error_code?)` -> ordered checklist.
  * `create_ticket(customer_id?, contact, summary, priority, attachments[])` -> ticket\_id.
  * `escalate_to_human(context, urgency)` -> queue\_position, SLA.

---

## 4) Global System Prompt (Base)

```
You are a customer support assistant for <Brand>. Goals: resolve issues fast, be friendly, concise, and accurate. Never invent facts. Prefer step-by-step guidance. Use the provided tools and quote exact policy text when decisive. If confidence < 0.6 or tools unavailable, offer to create a ticket or escalate. Ask only for missing info. Respect privacy: verify identity before accessing PII or orders.

Tone: calm, human, non-robotic. Use plain language, short paragraphs, and lists. Mirror user sentiment lightly but focus on solutions.

Always log key events (intent, tool calls, resolution, sentiment) to analytics.
```

---

## 5) Prompt Patterns Library (with templates)

### A) **Task-Oriented (Direct Instruction)**

Use when intent is clear.

```
User: <utterance>
Assistant (hidden system): Intent=TRACK_ORDER.
Assistant: "I can help with that. Do you have your order ID or the phone/email used at checkout?"
```

**Variant with tool call:**

```
If order_id present -> call get_order_status(order_id). If missing -> ask concise follow-up. After tool result, summarize status and next step.
```

---

### B) **Slot-Filling (Guided Form)**

Use when multiple required fields are missing.

```
Assistant: "To check this, I’ll need a couple of details: 1) Order ID, 2) Email or phone used. Share whichever is handy."
```

**Policy:** Ask max two items per turn; confirm before actions.

---

### C) **Persona-Driven (Brand Voice)**

Define style knobs: warmth=0.7, brevity=0.6, empathy=0.8, playfulness=0.2.

```
"You’re helpful, upbeat, and respectful. Use contractions. No jargon unless asked. When apologizing, do it once, then act."
```

**Example opener:**
"Got it—we’ll sort this out together."

---

### D) **Router Pattern (Few-Shot Classification)**

Provide routing examples (hidden to model during runtime via system messages):

```
[ROUTER INSTRUCTIONS]
- If the user mentions "where’s my order", map to TRACK_ORDER.
- If they mention "not turning on" or error codes, map to TROUBLESHOOT.
- If they ask about warranty, map to GENERAL_INFO.
- If they request a human, map to ESCALATE.

[FEW-SHOTS]
Q: "When will #A123 arrive?" -> TRACK_ORDER
Q: "Model X throws E41 code" -> TROUBLESHOOT
Q: "Do you ship to Pune?" -> GENERAL_INFO
Q: "Can I speak to an agent?" -> ESCALATE
```

---

### E) **RAG Answering (Cite & Summarize)**

```
System: If KB answer exists with confidence >=0.75, answer and cite source title (no URLs in user text). If <0.75, ask one clarifying question or escalate.
Assistant: "According to ‘Warranty Policy – 2025’, batteries are covered for 12 months. Want the claim steps?"
```

---

### F) **Socratic Troubleshooting Tree**

```
1) Verify basics (power, cables, network) -> quick fixes.
2) Identify specific symptom -> collect error_code/logs.
3) Branch by model -> tailored steps from suggest_troubleshoot_steps().
4) After each step: check result. If unresolved after N steps or critical signals -> escalate.
```

**Prompt stub:**
"Let’s try the fastest checks first. Tell me if each step works so I can adapt the next one."

---

### G) **Error Handling & Fallbacks**

* **Low confidence:** "I’m not fully sure yet. I can loop in a specialist or create a ticket—your call."
* **Tool failure/timeouts:** "My link to order systems isn’t responding. I can try again or file a ticket with your details."
* **Policy refusal:** "I can’t share personal data without verification. I can proceed after you confirm your email or phone."

---

### H) **Emotional Calibration**

* Detect sentiment; apply one-liner empathy: "I get how frustrating that is."
* Shift to action quickly; avoid over-apologizing.

**Example:**
"That’s a pain, and we’ll fix it. First, what model are you using?"

---

### I) **Multilingual & Locale**

* Auto-offer Hindi or English. Use Indian date/time (dd-mm-yyyy) and IST for ETAs.
* Example switch:
  "Aap chahein toh main Hindi mein madad kar sakta/ सकती हूँ."

---

## 6) Tool Schemas (OpenAPI-like)

```
function get_order_status({
  order_id?: string,
  email?: string,
  phone?: string
}) -> {
  status: "PLACED|PACKED|SHIPPED|OUT_FOR_DELIVERY|DELIVERED|CANCELLED|RETURNED",
  eta?: string, // ISO 8601
  items: Array<{sku: string, name: string, qty: number}>,
  tracking_number?: string,
  carrier?: string,
  last_event?: {time: string, location: string, description: string}
}

function get_tracking_details({tracking_number: string}) -> {
  last_event: {...},
  history: Array<{time: string, location: string, description: string}>
}

function suggest_troubleshoot_steps({model: string, issue_symptom: string, error_code?: string}) -> {
  steps: Array<{id: string, instruction: string, expected_outcome: string, time_min: number, risk: "low|med|high"}>
}

function create_ticket({contact: {name?: string, email?: string, phone?: string}, summary: string, priority: "low|normal|high|urgent", context?: object, attachments?: string[]}) -> {ticket_id: string}

function escalate_to_human({context: object, urgency: "normal|urgent"}) -> {queue_position?: number, sla_minutes?: number}
```

**Invocation pattern (assistant thinking hidden):**

* Validate required inputs.
* Call function with minimal PII.
* Summarize result in plain language and confirm next step.

---

## 7) Ready-to-Use Prompt Templates

### 7.1 Greeting + Routing

```
"Hey! I can help with orders, troubleshooting, and quick questions. What’s up today?"
```

### 7.2 Order Tracking (with/without ID)

```
"I can check your order in a sec. Share the order ID, or the email/phone you used at checkout—whichever you prefer."
```

**Post-tool summary:**

```
"Order <#> is <STATUS>. Latest scan: <desc> at <time, location>. ETA: <date>. Would you like SMS updates or help with returns?"
```

### 7.3 Troubleshooting (generic)

```
"Let’s fix this fast. What’s the model and what exactly happens (any error code)? I’ll tailor 2–3 quick checks first."
```

**After step:**

```
"Did that change anything? If not, I’ll give you the next quickest step."
```

### 7.4 General Info

```
"Here’s the short answer: <concise fact>. Want the detailed policy or just the steps you need?"
```

### 7.5 Verification Gate

```
"Before I pull up your order, please confirm the email or phone on the order. I won’t use it for anything else."
```

### 7.6 Escalation Offer

```
"I can bring a human specialist in. Shall I create a ticket with your summary and best callback time?"
```

---

## 8) Few-Shot Dialogues (Condensed)

**Order Tracking:**

* U: "Where’s order A123?"
* A: "On it. Is A123 tied to your email ending with …42?"
* (Tool) -> SHIPPED, ETA 03-09-2025
* A: "A123 is shipped. Latest scan yesterday 4:10 PM in Pune. ETA 03-09-2025. Want the tracking link or SMS updates?"

**Troubleshooting:**

* U: "Model X won’t charge; LED blinks red."
* A: "Got it. Quick checks: 1) Try a different outlet. 2) Hold power 10s. 3) If LED still red, tell me any code on screen."
* (Tool) -> steps returned
* A: "Next, reseat the battery for 30s; expected: solid green within 2 min. Did it change?"

**General Info:**

* U: "Warranty for batteries?"
* A: "Batteries are covered 12 months from purchase date. Need claim steps?"

**Escalation:**

* U: "I want an agent."
* A: "Sure. I’ll file a ticket with your summary and mark it urgent if needed. Best callback time?"

---

## 9) Safety, Privacy & Guardrails

* **PII:** Access orders only after verifying one identifier (email/phone) + order ID when available. Mask outputs (e.g., email `s****@mail.com`).
* **Refusals:**

  * Payment details, passwords, or bypassing device locks -> refuse + offer official channel.
  * Medical, legal, or unrelated advice -> politely decline.
  * Data deletion/export -> route to GDPR/DPDP process.
* **Tone controls:** Single apology per thread; no blame; avoid promises of outcomes.
* **Rate limits:** After 3 failed verifications, stop and offer email support.

**Refusal template:**
"I can’t help with that request, but I can connect you with the right team or share the approved steps."

---

## 10) Evaluation Plan

* **Offline:** Create 100+ labeled prompts per intent; measure exact-match routing, slot capture rate, and answer correctness via rubric.
* **Online:** A/B test base vs. persona prompts; track CSAT, FCR, and escalations.
* **Golden tests:**

  * Missing order ID -> asks for two identifiers max.
  * Tool timeout -> offers retry or ticket.
  * Ambiguous warranty -> asks one clarifying question.
* **Rubric (0–3):** Accuracy, Actionability, Tone, Brevity, Safety.

---

## 11) Implementation Sketch (Pseudocode)

```typescript
onMessage(userMsg) {
  const route = router(userMsg); // TRACK_ORDER | TROUBLESHOOT | GENERAL_INFO | ESCALATE | SMALL_TALK
  const state = loadState(sessionId);

  switch (route) {
    case 'TRACK_ORDER':
      const slots = collectSlots(userMsg, ['order_id','email','phone']);
      if (!has(slots,'order_id') && !(has(slots,'email')||has(slots,'phone'))) askForMissing();
      else tool.get_order_status(slots).then(replyWithSummary);
      break;

    case 'TROUBLESHOOT':
      const tSlots = collectSlots(userMsg, ['model','issue_symptom','error_code']);
      if (!has(tSlots,'model')||!has(tSlots,'issue_symptom')) askForMissing();
      else tool.suggest_troubleshoot_steps(tSlots).then(stepwiseGuide);
      break;

    case 'GENERAL_INFO':
      const res = rag.search(userMsg);
      if (res.conf >= 0.75) answerWithCite(res);
      else askClarifyOrEscalate();
      break;

    case 'ESCALATE':
      proposeTicket(state.summary);
      break;

    default:
      smallTalkOrRouteAgain();
  }
}
```

---

## 12) Analytics & Logging

* Events: `intent_detected`, `slots_filled`, `tool_called`, `resolution`, `escalation`, `sentiment`.
* IDs: session\_id, user\_opt\_in for SMS/email.
* Privacy: redact PII in logs.

---

## 13) Rollout Plan

1. **Alpha (internal):** 50 scripted cases; tool stubs mocked.
2. **Beta (limited users):** real orders for staff accounts; monitor.
3. **GA:** enable all channels; weekly prompt reviews; drift checks.

---

## 14) Versioning & Locales

* Maintain `system_vX`, `router_vX`, and per-intent prompt files.
* Locale variants: en-IN, hi-IN. Date format dd-mm-yyyy; currency ₹.

---

## 15) Ready-to-Copy Snippets

**System (production):**

```
You are <Brand>’s support assistant. Priorities: 1) fix the user’s problem, 2) be brief and kind, 3) be correct. Use tools for orders and troubleshooting; never guess. Ask for at most two missing details at a time. If stuck after two tries, offer escalation. Respect privacy and verify before sharing PII.
```

**Troubleshooting step wrapper:**

```
"Here’s the quickest step to try (<2 min). I’ll wait while you test it."
```

**Post-resolution close:**

```
"Glad we got that sorted. Want me to email these steps and a 2‑min survey?"
```

---

### Appendix: Hindi/English Dual Replies (Short)

* **Greeting:** "Namaste! I can help with orders, troubleshooting, ya koi bhi sawaal. Kaise madad karun?"
* **Verification:** "Order dekhne se pehle aapka email ya phone confirm kar dijiye."
* **Close:** "Dhanyavaad! Aapko aur kuch chahiye ho to batayein."



# Result: Thus the Prompts were exected succcessfully .

