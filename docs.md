# cs agent — detailed documentation

this document explains the full urvann customer support (cs) agent system in detail. for a quick visual overview, see [readme.md](readme.md).

**architecture diagram files:** [architecture-detailed.png](assets/architecture-detailed.png) · [architecture-detailed.mmd](assets/architecture-detailed.mmd) · [architecture-detailed.svg](assets/architecture-detailed.svg)

---

## table of contents

1. [overview](#overview)
2. [customer layer](#customer-layer)
3. [cs agent layer (new)](#cs-agent-layer-new)
4. [support tool apis (new)](#support-tool-apis-new)
5. [genie 2.0 (existing)](#genie-20-existing)
6. [storehippo (existing)](#storehippo-existing)
7. [customer intents and data routing](#customer-intents-and-data-routing)
8. [end-to-end conversation flow](#end-to-end-conversation-flow)
9. [authentication and session handling](#authentication-and-session-handling)
10. [data models reference](#data-models-reference)
11. [escalation and human handoff](#escalation-and-human-handoff)
12. [deployment and integration notes](#deployment-and-integration-notes)

---

## overview

the cs agent is an ai-powered customer support system for urvann, a plant and gardening e-commerce platform. when a customer needs help, they open support from the main urvann app and land in a dedicated chat experience at `support.urvann.com/chat`.

the system is designed in layers:

- **customer layer** — where the user starts (main app → cs agent app)
- **cs agent layer (new)** — the brain: chat api, llm orchestrator, conversation memory, policy knowledge base
- **support tool apis (new)** — structured functions the llm can call to fetch real data or take action
- **genie 2.0 (existing)** — urvann's internal order/tracking/refund backend backed by mongodb
- **storehippo (existing)** — the commerce platform (products, orders, returns, addresses)

the llm does not guess order status or refund details. it identifies what the customer wants, calls the right tool(s), and composes a reply from real data. for policy questions (refund rules, plant care), it reads from the policy knowledge base.

---

## customer layer

### urvann main app

the main urvann app handles shopping, order placement, and user profile management. when a customer taps **support** (typically from an order screen or profile), the app:

1. generates or reuses an auth token for the logged-in user
2. passes the current `txn_id` (transaction / order id) if the user opened support from an order context
3. deep-links or redirects to the cs agent app

### cs agent app

hosted at `support.urvann.com/chat`. this is a lightweight chat ui that:

- receives the token and optional `txn_id` from the main app
- opens a websocket or rest connection to the chat api
- renders messages from the customer and the agent
- shows typing indicators, error states, and escalation prompts when needed

**entry parameters**

| parameter | source | purpose |
|-----------|--------|---------|
| `token` | urvann main app | proves the user is authenticated |
| `txn_id` | urvann main app (optional) | pre-loads context for "this order" questions |

if `txn_id` is present, the agent can skip "which order?" clarification for order-specific queries.

---

## cs agent layer (new)

this is the core new infrastructure. it sits between the chat ui and all backend systems.

### customer auth

validates the token passed from the urvann main app before any chat message is processed.

- rejects expired or invalid tokens with a clear error to re-login in the main app
- extracts `user_id` and any embedded claims from the token
- attaches authenticated identity to the conversation session

auth runs on every new session and can be re-validated on sensitive tool calls (e.g. address change, ticket creation).

### chat api

**endpoint:** `post /support/chat`

handles inbound customer messages and outbound agent replies. supports:

- **rest** — simple request/response for each message
- **websocket** — persistent connection for real-time streaming replies

responsibilities:

1. receive message + session id
2. run auth check
3. pass message to the llm orchestrator
4. persist message to conversation store
5. return agent reply (or stream tokens)

### llm orchestrator

the orchestrator is the decision engine. for each customer message it:

1. **loads context** — recent conversation history from conversation store, optional pre-loaded `txn_id`
2. **classifies intent** — what is the customer asking? (tracking, refund, stock, care tips, etc.)
3. **selects tool(s)** — maps intent to one or more support tool api calls
4. **executes tools** — calls genie 2.0 and/or storehippo via tool wrappers
5. **generates reply** — uses tool results + policy kb to write a natural language response
6. **decides next step** — answer, ask clarifying question, create ticket, or escalate to human

the orchestrator follows a strict **intent → tool → reply** pattern. it should not fabricate order numbers, tracking statuses, or refund amounts.

### conversation store

persistent storage for chat sessions.

**stores:**

- `session_id` — unique per support conversation
- `user_id` — linked to authenticated customer
- `txn_id` — optional order context
- `messages[]` — full transcript (role, content, timestamp)
- `metadata` — intent history, tools called, escalation flags

used for:

- multi-turn context ("what about the other item in that order?")
- audit trail for support quality review
- handoff context when escalating to a human agent

### policy kb (knowledge base)

a curated knowledge base separate from live transactional data.

**contains:**

- refund policy rules (eligibility windows, partial vs full refund, damaged plant handling)
- return and replacement procedures
- plant care faqs (watering, sunlight, repotting, seasonal tips by plant type)
- delivery and attempt policies (`attemptno`, re-delivery rules)
- escalation criteria (when the bot should not auto-resolve)

the llm queries the kb for questions that do not require live order data, such as "how often should i water my monstera?" or "what is your refund policy for dead plants?"

---

## support tool apis (new)

these are structured functions exposed to the llm orchestrator. each tool wraps calls to genie 2.0 and/or storehippo and returns normalized json the llm can read.

### get_customer_orders

**purpose:** list all orders for the authenticated customer.

**calls:** genie 2.0 existing apis

**returns:** array of orders with `txn_id`, status, date, item summary

**used when:** customer asks "show my orders", "what did i order recently?", or has not specified which order

---

### get_order_details

**purpose:** full details for a single order.

**calls:** genie 2.0 `orderdetails`

**returns:** items[], shipping_address, payment status, rider info, support flags

**used when:** customer asks "what is in my order?", "what address is this going to?"

**key fields from genie:**

- `txn_id`
- `status`
- `items[]`
- `shipping_address`
- `user_id`
- `on_customer_support`

---

### get_order_tracking

**purpose:** live delivery tracking for an order.

**calls:** genie 2.0 `trackingroutes`

**returns:** tracker[] timeline, current status, rider_name, estimated delivery

**used when:** customer asks "where is my order?", "when will it arrive?", "who is delivering?"

**key fields:**

- `tracker[]` — status checkpoints with timestamps
- `rider_name`
- `attemptno` — delivery attempt count

---

### get_refund_status

**purpose:** refund progress for a return or cancellation.

**calls:** genie 2.0 (refundlog) and storehippo (`ms.returns`, `ms.refunds`)

**returns:** rma_number, refund_amount, refund_mode, status, expected timeline

**used when:** customer asks "where is my refund?", "when will i get my money back?"

genie holds internal refund logs; storehippo holds the commerce-side return/refund records. this tool merges both for a complete picture.

---

### check_product_stock

**purpose:** check if a plant/product is in stock.

**calls:** storehippo `ms.products`

**returns:** sku, inventory count, publish status, price

**used when:** customer asks "is the snake plant available?", "can i order this again?"

---

### search_products

**purpose:** find products by name, category, or keyword.

**calls:** storehippo `ms.products`

**returns:** matching products with sku, price, inventory, description snippet

**used when:** customer asks "do you have indoor plants under 500?", "show me low maintenance plants"

---

### create_ticket

**purpose:** log a support issue when the bot cannot auto-resolve.

**calls:** genie 2.0 `issueroutes`

**returns:** ticket id, confirmation message

**used when:** complex issue, customer explicitly requests follow-up, or policy requires human review

sets `on_customer_support` flag on the order in genie when applicable.

---

### escalate_to_human

**purpose:** transfer the conversation to a live human support agent.

**calls:** genie 2.0 + internal escalation queue

**returns:** queue position or agent assignment confirmation

**used when:**

- customer explicitly asks for a human
- sentiment is very negative after multiple failed attempts
- issue type is outside bot scope (legal, fraud, bulk b2b)
- policy kb marks the scenario as human-only

passes full conversation transcript from conversation store to the human agent.

---

## genie 2.0 (existing)

genie 2.0 is urvann's existing internal backend for order operations and support workflows. the cs agent does not replace genie — it consumes genie's apis through the tool layer.

### existing apis

| api route | purpose |
|-----------|---------|
| `trackingroutes` | order delivery tracking and rider updates |
| `issueroutes` | create and manage support issues/tickets |
| `orderdetails` | full order payload for a given txn_id |

### mongodb — store_hippo.orders

primary order collection. key fields used by the cs agent:

| field | description |
|-------|-------------|
| `txn_id` | unique transaction / order identifier |
| `status` | current order status (confirmed, dispatched, delivered, etc.) |
| `items[]` | line items with product names, quantities, skus |
| `tracker[]` | delivery milestone array |
| `rider_name` | assigned delivery rider |
| `shipping_address` | delivery address object |
| `user_id` | customer identifier |
| `on_customer_support` | boolean flag — order is under active support |
| `attemptno` | number of delivery attempts made |

### refundlog

internal refund tracking collection:

| field | description |
|-------|-------------|
| `refund_amount` | amount to be or already refunded |
| `rma_number` | return merchandise authorization number |
| `refund_mode` | original payment method, wallet, bank transfer, etc. |
| `status` | pending, processed, failed |

---

## storehippo (existing)

storehippo is urvann's e-commerce platform. the cs agent reads from (and in some cases writes to) storehippo for product and commerce data.

**base urls:** `urvann.com`, `urvann.storehippo.com`

### ms.products

product catalog:

- `sku` — stock keeping unit
- `price` — current selling price
- `inventory` — available stock count
- `publish` — whether product is live on site
- `description` — product details for search and care context

### ms.orders

commerce order records:

- `financial_status` — paid, pending, refunded
- `payment` — payment method and transaction references

### ms.returns + ms.refunds

return and refund lifecycle:

- `rma_number` — links return to refund
- refund status at commerce platform level

used alongside genie refundlog for `get_refund_status`.

### ms.user_addresses

customer saved addresses. used when customer asks to change delivery address (subject to order status — cannot change after dispatch in most cases).

---

## customer intents and data routing

the diagram maps common customer questions to the systems that answer them:

| customer question | primary data source | tool(s) used |
|-------------------|--------------------|--------------|
| where is my order? | genie 2.0 | `get_order_tracking` |
| what did i order? | genie 2.0 | `get_order_details`, `get_customer_orders` |
| where is my refund? | genie 2.0 + storehippo | `get_refund_status` |
| is this plant in stock? | storehippo | `check_product_stock`, `search_products` |
| change my address | storehippo | order status check → address update flow |
| plant care tips | policy kb | kb retrieval (no tool call to genie/storehippo) |

### routing logic summary

```
customer message
    → llm classifies intent
        → needs live order data?     → genie 2.0 tools
        → needs product/stock data?  → storehippo tools
        → needs policy/care info?    → policy kb
        → needs human?               → escalate_to_human / create_ticket
    → llm composes reply from results
```

---

## end-to-end conversation flow

### example: "where is my order?"

1. customer taps support in urvann app on order screen → cs agent app opens with `token` + `txn_id`
2. cs agent app connects to chat api via websocket
3. customer sends: "where is my order?"
4. chat api validates token via customer auth
5. llm orchestrator loads session; sees `txn_id` already in context
6. orchestrator calls `get_order_tracking(txn_id)`
7. tool calls genie 2.0 `trackingroutes` → reads `tracker[]`, `rider_name`, `attemptno` from mongodb
8. orchestrator receives structured tracking data
9. llm generates reply: "your order is out for delivery. rider rajesh will arrive today by 6 pm. this is delivery attempt 1."
10. reply streamed to cs agent app; both messages saved to conversation store

### example: "how do i care for my money plant?"

1. customer opens support without a specific order context
2. customer sends: "how do i care for my money plant?"
3. llm classifies intent as plant care (kb query, not transactional)
4. orchestrator queries policy kb for money plant care guidelines
5. llm generates reply from kb content — watering frequency, light requirements, common issues
6. no genie or storehippo call needed

### example: "i want to speak to someone"

1. customer sends escalation request
2. llm recognizes explicit human request
3. orchestrator calls `escalate_to_human(session_id, user_id, txn_id)`
4. full transcript attached from conversation store
5. customer sees: "connecting you to a support agent. estimated wait: 3 minutes."
6. human agent receives context in their queue tool

---

## authentication and session handling

### token flow

```
urvann main app (logged-in user)
    → generates signed token with user_id + expiry
    → passes token to cs agent app on support open
    → cs agent app sends token on every chat api request
    → customer auth middleware validates signature + expiry
    → user_id attached to session
```

### session lifecycle

| event | action |
|-------|--------|
| first message | create session in conversation store, bind user_id |
| subsequent messages | load session history for context window |
| session timeout (e.g. 30 min idle) | session marked inactive; new session on next message |
| escalation | session flagged; human agent added as participant |
| resolution | session closed; transcript retained for audit |

### security notes

- tokens must not be logged in plain text
- tool calls always scoped to authenticated `user_id` — a customer cannot query another user's orders
- sensitive actions (address change, ticket creation) may require re-validation

---

## data models reference

### conversation store — message object

```
{
  "session_id": "string",
  "user_id": "string",
  "txn_id": "string | null",
  "messages": [
    {
      "role": "user | assistant | system | tool",
      "content": "string",
      "timestamp": "iso8601",
      "tool_calls": []
    }
  ],
  "metadata": {
    "intents": [],
    "escalated": false,
    "created_at": "iso8601",
    "updated_at": "iso8601"
  }
}
```

### tool response — normalized shape

all tools return a consistent envelope so the llm can parse reliably:

```
{
  "tool": "get_order_tracking",
  "success": true,
  "data": { ... },
  "error": null
}
```

on failure:

```
{
  "tool": "get_order_tracking",
  "success": false,
  "data": null,
  "error": "order not found for txn_id: abc123"
}
```

---

## escalation and human handoff

the bot should escalate rather than guess when:

- tool calls return errors after retries
- customer repeats the same question 3+ times without satisfaction
- issue involves payment disputes above a threshold
- customer uses explicit escalation language ("manager", "complaint", "speak to human")
- policy kb marks scenario as human-required (e.g. bulk order cancellation)

**handoff package sent to human agent:**

- full conversation transcript
- user_id and contact info
- txn_id and order summary (if applicable)
- tools already called and their results (avoids customer repeating themselves)
- detected intent and sentiment score

---

## deployment and integration notes

### cs agent app

- url: `support.urvann.com/chat`
- communicates with chat api over rest and/or websocket
- should handle token expiry gracefully (redirect to main app login)

### chat api

- new service to be deployed alongside or in front of genie 2.0
- requires connection to: conversation store (db), policy kb (vector store or document db), llm provider api

### tool api layer

- can be implemented as functions within the orchestrator service or as a separate microservice
- each tool is a thin wrapper with input validation, auth scoping, and response normalization

### genie 2.0 integration

- no changes required to genie mongodb schema for read operations
- `issueroutes` and `on_customer_support` flag used for write operations (tickets, escalation)

### storehippo integration

- read access to `ms.products`, `ms.orders`, `ms.returns`, `ms.refunds`, `ms.user_addresses`
- write access needed only for address update flows (with status guards)

### policy kb

- can start as markdown documents ingested into a vector store
- should be owned by support/operations team with a regular update cadence
- version kb content so replies can be traced to a specific policy version

---

## glossary

| term | meaning |
|------|---------|
| `txn_id` | transaction id — urvann's primary order identifier |
| `rma_number` | return merchandise authorization — links a return to its refund |
| `attemptno` | number of delivery attempts for an order |
| `on_customer_support` | flag on an order indicating active support involvement |
| genie 2.0 | urvann internal order and support backend |
| storehippo | third-party e-commerce platform powering urvann's catalog and checkout |
| policy kb | knowledge base of support policies and plant care content |
| llm orchestrator | ai layer that routes intents to tools and composes replies |

---

*for the architecture diagram, see [readme.md](readme.md).*
