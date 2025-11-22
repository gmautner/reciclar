# Requirements

## Business Requirements

### Context

Reciclar is a WhatsApp group in which members announce used goods for sale. Group members usually post messages with a photo of the item and a description including its price.

Members can both post and buy items. The bidding process works as follows:

- The intested buyer replies to the post (or send a message right after the post) with the message "F1" (which means "Fila 1" or "primeiro da fila"), or similar messages like "Fila 1", "fila 1" etc.
- Other buyers can reply with follow-up messages like "F2/Fila 2", and "F3/Fila 3"
- If there is one buyer in the queue (only one who submitted "fila 1" or "F1"), after a set timeout agreed by the members (usually 5 minutes), the buyer is automatically the winner and the item is sold to them.
- If a second buyer enters the queue (submitted "fila 2" or "F2"), after a set timeout agreed by the members (usually 5 minutes), in case no other buyer enters the queue, the first buyer is the winner and the item is sold to them.
- If a third buyer enters the queue (submitted "fila 3" or "F3"), a "leilão" is launched. Then, any member in the group can reply with a bid. Each bid should be both bigger than the announced price and bigger than the previous bid.
- After a set timeout or fixed end time agreed by the members, the highest bidder is the winner and the item is sold to them.
- In any of the above cases, the seller should announce to the group that the item has been sold to the winner. There are different ways in which sellers use to announce the sale. For example:
  - A reply to the buyer's message with a message like "Pode pagar"
  - A special sticker recognized by the group as a confirmation of the sale
- The payment occurs outside the group.

### Goals

In our first release, we will not focus on the bidding process and let the members manage it manually. We will focus on the following goals:

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
