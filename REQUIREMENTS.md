# Reciclar - Application Requirements & Technical Specification

## 1. Project Overview

**Reciclar** is a web application that automates the management of a "Buy & Sell" WhatsApp group. It connects to a [WAHA](https://waha.devlike.pro/) instance to read messages, uses a minimal AI agent (only for parsing descriptions/prices from natural language), and deterministic logic to track sales flows (Queues, Auctions). The app presents a dashboard of closed sales.

## 2. Business Logic & AI Heuristics

### 2.1. Offer Identification (The "Parent-Child" Grouping Rule)

The system must correctly group multiple messages into a single "Offer" entity, especially when sellers post batch listings.

* **The Parent (New Offer):** Any message containing an **Image** + **Text with a Price** is a **New Offer**.
* **The Child (Detail Photo):** Any message containing an **Image** but **NO Price** is considered a detail photo of that Seller's **most recent Offer** (i.e., the most recent message from the same Seller containing an Image and Text with a Price).
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

### 2.3. Message Processing Strategy

The system uses a **minimal AI + deterministic logic** approach:

* **AI Agent (Simplified):** Only used for extracting natural language descriptions and prices from text:
  * Extract `description + price` from new offer messages
  * Extract `price only` from bid messages
  
* **Deterministic Logic:** All other processing is done through Python code:
  * **Queue Detection:** Regex pattern `(F|f)(ila)?\s*\d+` to extract queue numbers
  * **Seller Triggers:** Pattern matching for "Pode pagar", "Seu", "Fechado", "Leilão", "Valendo"
  * **Parent-Child Linking:** Messages with images but no price are linked to the seller's most recent offer
  * **State Machine:** Python code manages transitions (OPEN → QUEUE → AUCTION → PENDING → SOLD)
  * **Sticker Parsing:** For now, assume sticker content is deterministic (check for `message.type == 'sticker'` via WAHA API)

* **Price Formats Supported:** The AI agent should recognize:
  * `R$ 100` / `r$ 100` (Case insensitive, optional space)
  * `100,00` (Comma decimal)
  * `100 reais` (Suffix)
  * `R$100,00`
  * And any combination of the above

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

### 4.3. AI Layer (Minimalist Approach)

* **Library:** `instructor` (for Structured Output).
* **Models:** OpenAI (GPT-4o-mini) or Anthropic (Claude Haiku) - cheaper models sufficient for simple extraction.
* **Single Agent Purpose:** Extract descriptions and prices from natural language text.
  * **Input:** Message text content
  * **Output:** Either `(description, price)` OR `(price_only)` OR `None`
  * **Usage:** Only called when message contains text that may be an offer or bid
  
* **Everything Else is Deterministic:**
  * Queue detection via regex
  * State machine transitions via Python logic
  * Thread grouping via image presence + price detection
  * Seller trigger phrases via pattern matching

## 5. Data Models (Pydantic)

### 5.1. AI Extraction Models (Simplified)

These models are used ONLY for the AI agent's structured output:

```python
from typing import Optional, Literal
from pydantic import BaseModel, Field

class OfferExtraction(BaseModel):
    """AI extracts item description + price from text"""
    description: str = Field(description="Natural language description of the item being sold")
    price: float = Field(description="Price in BRL (Brazilian Reais)")

class BidExtraction(BaseModel):
    """AI extracts only a price from text (for auction bids)"""
    price: float = Field(description="Bid amount in BRL")

class TextExtraction(BaseModel):
    """Union type - AI returns one of these based on text content"""
    extraction_type: Literal["offer", "bid", "none"]
    offer: Optional[OfferExtraction] = None
    bid: Optional[BidExtraction] = None
```

### 5.2. Business Logic Models (Database/Application)

These models represent the actual data stored and processed:

```python
from enum import Enum
from typing import Optional, List
from pydantic import BaseModel

class OfferStatus(str, Enum):
    OPEN = "open"
    QUEUE = "queue"
    AUCTION = "auction"
    PENDING = "pending"
    SOLD = "sold"

class Offer(BaseModel):
    id: str
    description: str
    item_photo_url: str
    initial_price: float
    
    seller_name: str
    seller_phone: str
    buyer_name: Optional[str] = None
    buyer_phone: Optional[str] = None
    
    final_price: Optional[float] = None
    status: OfferStatus
    created_at: str
    closed_at: Optional[str] = None
    
    # Child messages (detail photos)
    detail_photo_urls: List[str] = []

class QueueEntry(BaseModel):
    """Tracks who entered the queue for an offer"""
    offer_id: str
    position: int  # 1, 2, 3...
    buyer_name: str
    buyer_phone: str
    timestamp: str

class Bid(BaseModel):
    """Tracks auction bids"""
    offer_id: str
    bidder_name: str
    bidder_phone: str
    amount: float
    timestamp: str
```

## 6. Docker & Deployment

* **Container:** Python 3.11-slim.
* **Server:** `uvicorn`.
* **Tooling:** Include `cmk` CLI.
* **Database:** Connect to external Postgres via standard Environment Variables (sourced from `.env`).
* **Deliverable:** `Dockerfile` and `README.md` with build/run instructions.
