# cs agent draft

architecture overview for the urvann customer support (cs) agent system.

## system architecture

```mermaid
flowchart TB
    subgraph customer["customer layer"]
        ua["urvann main app<br/>shopping - orders - profile"]
        csa["cs agent app<br/>support.urvann.com/chat"]
        ua -->|"tap support<br/>pass token + txn_id"| csa
    end
    subgraph agent["cs agent layer - new"]
        auth["customer auth<br/>validate token from urvann app"]
        chat["chat api<br/>post /support/chat"]
        llm["llm orchestrator<br/>intent to tool to reply"]
        conv[("conversation store<br/>sessions, messages")]
        kb[("policy kb<br/>refund rules, care faqs")]
        csa <-->|"rest / websocket"| chat
        chat --> auth
        chat --> llm
        llm --> conv
        llm --> kb
    end
    subgraph tools["support tool apis - new"]
        direction LR
        t1["get_customer_orders"]
        t2["get_order_details"]
        t3["get_order_tracking"]
        t4["get_refund_status"]
        t5["check_product_stock"]
        t6["search_products"]
        t7["create_ticket"]
        t8["escalate_to_human"]
    end
    llm --> tools
    subgraph genie["genie 2.0 - existing"]
        direction TB
        g_api["existing apis<br/>trackingroutes, issueroutes, orderdetails"]
        g_db[("mongodb store_hippo.orders<br/>txn_id, status, items[]<br/>tracker[], rider_name<br/>shipping_address, user_id<br/>on_customer_support, attemptno")]
        g_refund[("refundlog<br/>refund_amount, rma_number<br/>refund_mode, status")]
        g_api --> g_db
        g_api --> g_refund
    end
    subgraph sh["storehippo - existing"]
        direction TB
        sh_api["storehippo rest api<br/>urvann.com, urvann.storehippo.com"]
        sh_prod["ms.products<br/>sku, price, inventory<br/>publish, description"]
        sh_ord["ms.orders<br/>financial_status, payment"]
        sh_ret["ms.returns + ms.refunds<br/>rma_number, refund status"]
        sh_addr["ms.user_addresses<br/>address update"]
        sh_api --> sh_prod & sh_ord & sh_ret & sh_addr
    end
    t1 & t2 & t3 & t7 & t8 --> g_api
    t4 --> g_api & sh_api
    t5 & t6 --> sh_api
    subgraph intent["customer questions - data source"]
        direction LR
        i1["where is my order?"] --> genie
        i2["what did i order?"] --> genie
        i3["where is refund?"] --> genie & sh
        i4["is plant in stock?"] --> sh
        i5["change address"] --> sh
        i6["plant care tips"] --> kb
    end
    style customer fill:#e8f4fd,stroke:#2196F3
    style agent fill:#fff3e0,stroke:#FF9800
    style tools fill:#f3e5f5,stroke:#9C27B0
    style genie fill:#e8f5e9,stroke:#4CAF50
    style sh fill:#fce4ec,stroke:#E91E63
    style intent fill:#f5f5f5,stroke:#9E9E9E
```

## layers

| layer | description |
|-------|-------------|
| customer | urvann main app routes users to the cs agent app with auth token and transaction id |
| cs agent (new) | chat api, llm orchestrator, conversation store, and policy knowledge base |
| support tools (new) | tool apis for orders, tracking, refunds, stock, tickets, and escalation |
| genie 2.0 (existing) | order tracking, issues, and refund data via mongodb |
| storehippo (existing) | product catalog, orders, returns/refunds, and address management |
