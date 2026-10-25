# AI Quote Generator

A webhook for an HVAC business that turns a customer's quote request into an estimated price range and a short quote draft. OpenAI writes the estimate, and the owner approves it on Telegram before the customer gets an email. Every quote is saved to Supabase with its status, so the team can see what was sent, rejected, or still waiting.

## Architecture

```
Quote Request Webhook (POST /quote-request)
   → Validate Fields (name, email format, service, home size, location)
        ├── Invalid → Respond Invalid (400)
        └── Valid → AI Price Estimate (OpenAI, gpt-4o-mini)
                  → Parse AI Response (read JSON, map fields)
                  → Save Quote to Supabase (status: pending_review)
                       ├── Respond Success (quote ID, range, draft)
                       └── Telegram Review (owner approves or rejects)
                              → Approved?
                                   ├── Yes → Email Quote to Customer → Mark Quote Sent
                                   └── No  → Mark Quote Rejected
```

## Features

* Checks that name, email (with a valid format), service, home size, and location are all provided before any AI call
* Returns HTTP 400 with a clear message when fields are missing
* AI gives a rough USD price range and a 3 to 4 sentence quote draft, returned as JSON
* The owner reviews every quote on Telegram before the customer sees it
* Each quote is saved with a status: `pending_review`, then `sent` or `rejected`
* The webhook returns the quote ID and a "being reviewed" message right away. The price is not shown to the caller until the owner approves
* Rejected quotes are never emailed to the customer

## Tech Stack

* n8n 2.x (workflow automation, webhooks, human approval)
* OpenAI `gpt-4o-mini` (price estimate and draft)
* Supabase (quotes table)
* Telegram Bot API (owner approval)
* Gmail API (customer email)

## Setup

1. Create a Supabase table named `quotes` with these columns:

```
id | name | email | service | home_size | location | price_min | price_max | quote_draft | status
```

2. Import `ai_quote_generator.json` into n8n.
3. Add credentials: OpenAI, Supabase, Telegram, and Gmail OAuth2. Assign them to the matching nodes.
4. Replace `YOUR_TELEGRAM_CHAT_ID` in the Telegram Review node with the owner's chat ID.
5. Set up an error workflow that sends you an alert, and select it in the workflow settings as the **Error Workflow**.
6. Activate the workflow.
7. Send a test request, then approve or reject the review message on Telegram.

## Sample Request

```
POST https://YOUR_N8N_DOMAIN/webhook/quote-request
Content-Type: application/json

{
  "name": "Ali Raza",
  "email": "ali.test@example.com",
  "service": "AC installation",
  "home_size": "1500 sq ft",
  "location": "Karachi"
}
```

Expected result: a `quotes` row with status `pending_review`, a Telegram review message, and a response like:

```
{ "status": "pending_review", "quote_id": 12, "message": "Your quote is being reviewed. We will contact you shortly." }
```

After approval, the customer gets the email and the row changes to `sent`. After rejection, the row changes to `rejected` and no email is sent.

Missing a field returns HTTP 400 with `{"status": "invalid", "message": "Please fill all required fields."}`.

## Known Limits

* The price range is an AI estimate, not a site visit. Treat it as a rough guide. Add a range check in code if you want tighter limits.
* If the AI reply is not valid JSON, the run stops, the error workflow is notified, and the caller gets no response. Add a retry or a manual-review branch if you need a reply in that case.
* If the owner does not answer the Telegram review, the quote stays `pending_review`. Check the table regularly.
* The caller gets the "being reviewed" response before the owner decides. If you want the caller to receive the price, send it from the approval step (for example by email), not in the first response.

## Project Notes

* Built as part of my AI Automation portfolio.
* The model writes the estimate, but the fields that get saved and emailed are read from its JSON in code, so each field has a fixed place.
* The owner approves every quote, so the AI never sends a price to a customer on its own, and the price is not returned to the caller before approval.
* The AI model is `gpt-4o-mini`. Change the model in the AI Price Estimate node if you want another.
