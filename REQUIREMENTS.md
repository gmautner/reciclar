# Reciclar - Application Requirements & Technical Specification

## 1. Project Overview

**Reciclar** is a web application that automates the management of a "Buy & Sell" WhatsApp group. It connects to a [WAHA](https://waha.devlike.pro/) instance to read messages, uses AI agents to interpret complex sales flows (Queues, Auctions), and presents a dashboard of closed sales.

## 2. Business Logic & AI Heuristics

### 2.1. Offer Identification (The "Parent-Child" Grouping Rule)

The system must correctly group multiple messages into a single "Offer" entity, especially when sellers post batch listings.

* **The Parent (New Offer):** Any message containing an **Image** + **Text with a Price** is a **New Offer**.
* **The Child (Detail Photo):** Any message containing an **Image** but **NO Price** that appears within **2 minutes** of a "Parent" message by the *same Seller* is considered a detail photo of the *same* Offer.
* **The Stray:** An image without price that appears after the time window is ignored or treated as chatter.

### 2.2. The Buying Process (State Machine)

Each Offer moves through the following states:

1. **OPEN**: Initial state when listed.

2. **QUEUE (Fila)**:
   * **Trigger:** A user replies with `(F|f)(ila)?\s*\d+` (e.g., "F1", "Fila 1", "f2").
   * **Strictness:** Vague terms ("Quero", "Mio") must be **ignored**.
   * **Rule:** The first valid "F1" holder is the tentative winner.

3. **AUCTION (Leilão)**:
   * **Trigger:** Either a 3rd buyer enters the queue ("F3") OR the Seller explicitly announces "Leilão" / "Valendo".
   * **Unlinked Bids Rule:** During a "Hot Auction", bidders often send "naked numbers" (e.g., "550", "600") without using the Reply feature.
      * *Logic:* If a message contains only a number and has NO reply reference, associate it with the **most recent Offer currently in the AUCTION state**.

4. **AWAITING PAYMENT (Pendente)**:
   * **Trigger:** Seller sends a text like "Pode pagar", "Seu", "Fechado".
   * **Status:** The sale is locked but NOT final.

5. **SOLD (Vendido)**:
   * **Strict Trigger:** The Seller posts a specific **Confirmation Sticker**.
   * **Validation:** The system must detect the text *"VENDIDO OBRIGADA POR AJUDAR O Lar das Crianças"* inside the sticker image.

### 2.3. Parsing Rules

* **Price Regex:** The agent must identify prices in these formats:
  * `R$ 100` / `r$ 100` (Case insensitive, optional space)
  * `100,00` (Comma decimal)
  * `100 reais` (Suffix)
  * `R$100,00`
* **Sticker Handling:**
  * Detect `message.type == 'sticker'` via WAHA.
  * Send the sticker image (Base64/URL) to the AI Vision model to read the text.
  * Only trigger "SOLD" state if the text matches the specific charity string above.

## 3. Web Application Goals

### 3.1. Dashboard (Closed Orders)

The main view is a dashboard displaying a table of **Closed Orders**.

* **Columns Required:**
  * **Buyer:** Name and Phone (`+<country><number>`).
  * **Seller:** Name and Phone (`+<country><number>`).
  * **Item:** Photo thumbnail + Description.
  * **Price:** The *ultimate* price settled (auction price or list price).
  * **Date:** Timestamp of the sale.
* **Localization:** All UI text in Brazilian Portuguese.
* **Features:**
  * **Filters:** Buyer Name, Seller Name, Price Range, Date Range.
  * **Pagination:** Support for different page sizes.
  * **Export:** "Export to Excel" button (must respect currently active filters).

### 3.2. Authentication

* **Provider:** Google OAuth.
* **Role Management:**
  * The **first** user to sign up becomes the **Admin**.
  * Subsequent users are "Pending" until approved by the Admin.

## 4. Tech Stack

### 4.1. Core

* **Language:** Python 3.11+.
* **Web Framework:** FastAPI.
* **Database:** PostgreSQL + SQLModel (SQLAlchemy).
* **Frontend:** HTMX + Jinja2 Templates + Tailwind CSS + DaisyUI.
* **WhatsApp Gateway:** [WAHA](https://waha.devlike.pro/) (provided externally).

### 4.2. Frontend Architecture

Use the "HATEOAS-lite" approach:

```text
[User Action] -> HTMX Request -> FastAPI -> Render Jinja2 Partial -> HTMX Swap
```

### 4.3. AI Layer

* **Library:** `instructor` (for Structured Output).
* **Models:** OpenAI (GPT-4o) or Anthropic (Claude 3.5) for text/vision.
* **Agent Strategy:**
  1. **Thread Mapper:** Groups raw messages into "Offer Threads" (handling Parent/Child images).
  2. **Interaction Extractor:** Parses intent (Bid, Queue, Question) using the Data Models below.

## 5. Data Models (Pydantic)

Use these models to enforce structured data extraction:

```python
from enum import Enum
from typing import Optional, List
from pydantic import BaseModel, Field

class MessageIntent(str, Enum):
    NEW_OFFER = "new_offer"
    DETAIL_PHOTO = "detail_photo"    # Image w/o price, linked to previous offer
    QUEUE_ENTRY = "queue_entry"      # "F1", "Fila 2"
    BID = "bid"                      # "550", "600"
    QUESTION = "question"            # "Qual medida?"
    PAYMENT_REQUEST = "payment_request" # "Pode pagar"
    SALE_CONFIRMED = "sale_confirmed"   # Validated by Sticker
    CHATTER = "chatter"

class ExtractedInteraction(BaseModel):
    message_id: str
    reply_to_id: Optional[str]
    sender_name: str
    sender_phone: str
    timestamp: str
    intent: MessageIntent
    
    # Specifics based on intent
    price_detected: Optional[float] = Field(description="For Bids or New Offers")
    queue_number: Optional[int] = Field(description="1 for F1, 2 for F2...")
    sticker_text_detected: Optional[str]

class Offer(BaseModel):
    id: str
    child_message_ids: List[str] = []
    description: str
    item_photo_url: str
    
    seller_name: str
    seller_phone: str
    buyer_name: Optional[str]
    buyer_phone: Optional[str]
    
    final_price: Optional[float]
    status: str # OPEN, QUEUE, AUCTION, PENDING, SOLD
    created_at: str
    closed_at: Optional[str]
```

## 6. Docker & Deployment

* **Container:** Python 3.11-slim.
* **Server:** `uvicorn`.
* **Tooling:** Include `cmk` CLI.
* **Database:** Connect to external Postgres via standard Environment Variables (sourced from `.env`).
* **Deliverable:** `Dockerfile` and `README.md` with build/run instructions.
