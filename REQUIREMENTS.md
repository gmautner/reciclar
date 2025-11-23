# Requirements

## Business Requirements

### Outline

Reciclar is a WhatsApp group in which members offer used goods for sale. Group members usually post offers with a photo of the item and a description including its price.

Members can both offer and buy items. The bidding process works as follows:

- The intested buyer replies to the offer (or send a message right after the offer) with the message "F1" (which means "Fila 1" or "primeiro da fila"), or similar messages like "Fila 1", "fila 1" etc.
- Other buyers can reply with follow-up messages like "F2/Fila 2", and "F3/Fila 3"
- If there is one buyer in the queue (only one who submitted "fila 1" or "F1"), after a set timeout agreed by the members (usually 5 minutes), the buyer is considered the winner and the item is sold to them.
- If a second buyer enters the queue (submitted "fila 2" or "F2"), after a set timeout agreed by the members (usually 5 minutes), in case no other buyer enters the queue, the first buyer is considered the winner and the item is sold to them.
- If a third buyer enters the queue (submitted "fila 3" or "F3"), an auction is launched. Then, any member in the group can reply with a bid. Each bid should be both bigger than the announced price and bigger than the previous bid.
- After a set timeout or fixed end time agreed by the members, the highest bidder is considered the winner and the offered item is sold to them.
- In any of the above cases, the seller should announce to the group that the offered item has been sold to the winner. There are different ways in which sellers use to announce the sale. For example:
  - A reply to the buyer's message with a message like "Pode pagar"
  - A special sticker recognized by the group as a confirmation of the sale.
  - Other kinds of confirmations.
  - IMPORTANT: when announcing the sale, the seller may or may not mention who's the winner. If not mentioned, the winner should be inferred from the context according to the rules above.
- There is one second kind of process, usually used when the offer is of a higher value. In this case, the offer is described, and a statement that the offer will be auctioned is made. The statement may contain different rules like the timeframe for the auction, the starting price, the increment price, etc.
  - In such auctions, bidders send messages with the amount of the bid.
  - After the conditions for the auction conclusion are met, the seller announces the conclusion of the auction.
  - IMPORTANT: when announcing the conclusion of the auction, the seller may or may not mention who's the winner. If not mentioned, the winner AND the final price should be inferred from the context according to the rules above.
- The payment occurs outside the group and is not our concern.

### Goals

In our first release, we will focus on the following goals:

- Create a dashboard for viewing closed orders. Each record should contain:

  - Buyer's name and phone number in format `+<country code><phone number>`
  - Seller's name and phone number in format `+<country code><phone number>`
  - The item's photo
  - The item's description
  - The ultimate price settled by the auction
  - The date and time of the sale

## Tech Stack

A [WAHA](https://waha.devlike.pro/) instance will be used to enable the interface with the WhatsApp group and will be externally provided to us.

### Framework

Use Python FastAPI with HTMX, Tailwind CSS and DaisyUI.

For the database, use PostgreSQL. Credentials should either be sourced from an `.env` file (make sure to exclude it in `.gitignore`) or from environment variables.

Pages (or partials) should be retrieved using this schematic approach:

```text
[User Click]
     ↓
 HTMX sends AJAX (hx-post, hx-get)
     ↓
 FastAPI endpoint
     ↓
 SQLModel session → DB
     ↓
 Render Jinja2 partial → HTML
     ↓
 HTMX swaps snippet into DOM
```

Make sure to use up to date versions of the libraries and frontend components.

### Authentication

Users may sign up and login using a Google account.

The first user to sign up should be the admin. After that, any subsequent user that signs up will be subject to the admin's approval to access the system.

## Docker packaging

Run with `uvicorn`. No need to package Postgres as it will be provided externally. Include the `cmk` CLI in the Docker image.

Provide instructions in the README on how to build and run the Docker image, including how to set the environment variables and local Postgres container.
